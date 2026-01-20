# Detail Pointing & Label System

This document describes the dual-hand pointing system where one hand grabs an object while the other hand points at specific details to receive contextual labels and explanations.

## Overview

The system enables users to:
1. Grab an object with their **LEFT hand**
2. Point at specific details/parts of that object with their **RIGHT hand** (extended index finger)
3. Receive real-time labels and explanations that spawn at the fingertip position
4. Have those labels track with the grabbed object

---

## Complete User Journey

```
1. LEFT HAND GRABS OBJECT
   - HandGrabTrigger detects grab via Gemini vision
   - Twin object spawns for visual feedback
   - Object label hides, sphere mesh hides
   - Object follows hand via DualTargetLazyFollow
   ↓
2. RIGHT HAND APPROACHES (within maxHandProximityDistance: 20cm default)
   - CheckHandsProximity() monitors wrist distance
   - Both hands must be tracked
   ↓
3. RIGHT INDEX FINGER EXTENDS (pointing gesture)
   - System checks angle between index finger direction and hand direction
   - Angle must be < 25 degrees to be considered "pointing"
   ↓
4. POINTING DETECTED (both conditions met)
   - pointingPlane GameObject activates
   - Gemini vision analyzes current camera frame
   - Identifies which part user is pointing at
   ↓
5. LABEL SPAWNS AT FINGERTIP
   - Position: fingerTipPose.position + (Vector3.up * planeUpOffset)
   - Uses XRHandJointID.IndexTip for precise tracking
   ↓
6. LABEL REPARENTED TO HOLDING HAND
   - pointingPlane.transform.SetParent(holdingHand.transform)
   - Stores relativePosition = localPosition for tracking
   - DualTargetLazyFollow added for camera-facing rotation
   ↓
7. LABEL DISPLAYS PART INFO
   - pointingPlaneText shows the part name
   - descriptionText shows the explanation
   - Questions can be generated for the specific part
   ↓
8. WHEN POINTING ENDS
   - pointingPlane.SetActive(false)
   - relativePosition reset to Vector3.zero
   ↓
9. WHEN OBJECT RELEASED
   - Label system resets
   - Object returns to scene with visible label/sphere
```

---

## Key Files & Code Locations

### Primary File: `Assets/SphereToggleScript.cs`

| Feature | Lines | Description |
|---------|-------|-------------|
| `pointingPlane` declaration | ~127 | GameObject reference for the pointing label |
| `pointingPlaneText` | ~130 | TextMeshPro for displaying part name |
| `maxHandProximityDistance` | ~139 | Distance threshold (default 0.2m / 20cm) |
| `planeUpOffset` | ~133 | Offset above fingertip (default 0.02m) |
| `CheckHandsProximity()` | ~1217-1273 | Detects dual-hand pointing configuration |
| `UpdateObjectDescriptionRoutine()` | ~932-1214 | Main coroutine for continuous pointing detection |
| Label spawn at fingertip | ~1045 | `pointingPlane.transform.position = fingerTipPose.position + (Vector3.up * planeUpOffset)` |
| Reparent to holding hand | ~1048-1051 | `pointingPlane.transform.SetParent(holdingHand.transform)` |
| DualTargetLazyFollow config | ~1057-1087 | Configures rotation to face camera |
| Part identification via Gemini | ~966-986 | Prompt asking Gemini to identify pointed part |
| `OnPointingStateHandler()` | ~202-270 | Handles pointing state changes |

### Supporting File: `Assets/Script/Granularity Lv3/HandGrabTrigger.cs`

| Feature | Lines | Description |
|---------|-------|-------------|
| `OnGeminiGrabbingDetected()` | ~339-463 | Handles grab detection |
| `ManualGrabAnchor()` | ~471-564 | Manual grab initiation |
| `ConfigureDualTargetLazyFollow()` | ~566-592 | Sets up object following hand |
| `ReleaseAnchor()` | ~868-994 | Handles anchor release |
| Twin object management | ~435-443, ~848-863 | Spawns/destroys twin object |

### Supporting File: `Assets/Script/MyHandTracking.cs`

| Feature | Description |
|---------|-------------|
| Hand joint tracking | Provides XRHand data for both hands |
| `m_SpawnedLeftHand` / `m_SpawnedRightHand` | References to spawned hand GameObjects |

---

## How Pointing Detection Works

### Step 1: Hand Proximity Check

From `CheckHandsProximity()` in SphereToggleScript.cs:

```csharp
// Get wrist positions from both hands
Pose leftWristPose, rightWristPose;
bool leftHandTracked = handSubsystem.leftHand.GetJoint(XRHandJointID.Wrist).TryGetPose(out leftWristPose);
bool rightHandTracked = handSubsystem.rightHand.GetJoint(XRHandJointID.Wrist).TryGetPose(out rightWristPose);

// Calculate distance between wrists
float wristDistance = Vector3.Distance(leftWristPose.position, rightWristPose.position);
bool handsInProximity = wristDistance <= maxHandProximityDistance; // 20cm default
```

### Step 2: Pointing Gesture Detection

```csharp
// Get index finger joint positions
Pose rightIndexTipPose, rightIndexProximalPose;
handSubsystem.rightHand.GetJoint(XRHandJointID.IndexTip).TryGetPose(out rightIndexTipPose);
handSubsystem.rightHand.GetJoint(XRHandJointID.IndexProximal).TryGetPose(out rightIndexProximalPose);

// Calculate finger direction (tip - base)
Vector3 indexDirection = (rightIndexTipPose.position - rightIndexProximalPose.position).normalized;

// Calculate hand direction (fingertip - wrist)
Vector3 handDirection = (rightIndexTipPose.position - rightWristPose.position).normalized;

// Check angle between vectors - small angle means finger is extended/pointing
float angle = Vector3.Angle(indexDirection, handDirection);
bool isRightHandPointing = angle < 25f; // degrees threshold
```

---

## Label Spawn & Parenting Flow

### Initial Positioning

```csharp
// Position at RIGHT index fingertip with slight vertical offset
pointingPlane.transform.position = fingerTipPose.position + (Vector3.up * planeUpOffset);
```

### Reparenting to Holding Hand

```csharp
// Parent to LEFT hand (holding hand) so label follows grabbed object
if (!pointingPlane.transform.IsChildOf(holdingHand.transform))
{
    pointingPlane.transform.SetParent(holdingHand.transform);
}

// Store relative position for reference
relativePosition = pointingPlane.transform.localPosition;
```

### Camera-Facing Rotation

```csharp
// Add DualTargetLazyFollow for billboard effect
var dualLazyFollow = pointingPlane.AddComponent<DualTargetLazyFollow>();

// Configure: NO position following (stays at fingertip), YES rotation following (faces camera)
dualLazyFollow.positionFollowMode = LazyFollow.PositionFollowMode.None;
dualLazyFollow.rotationFollowMode = LazyFollow.RotationFollowMode.LookAt;
dualLazyFollow.rotationTarget = Camera.main.transform;

// Smooth rotation parameters
dualLazyFollow.movementSpeed = 15f;
dualLazyFollow.minAngleAllowed = 3f;
dualLazyFollow.maxAngleAllowed = 15f;
```

---

## Part Identification via Gemini

The system sends the current camera frame to Gemini with this prompt:

```csharp
string prompt = $@"
You are analyzing a {labelContent} in real-time.
Scene context: {currentSceneContext}
Task context: {currentTaskContext}

Based on the current image and considering the previous observations:
1. Describe any NEW details or changes you notice about the object
2. Focus on aspects not mentioned before
3. Only describe the part where the user is currently pointing at
4. Consider the object's current state, position, and interaction with the environment
5. If you don't see any new information, respond with: {{""part"": ""none"", ""description"": ""No new observations.""}}
6. If the user is not pointing at the object, respond with: {{""part"": ""none"", ""description"": ""Not being pointed at.""}}
7. The user pointing at the object is because they don't fully understand this part. So explain it in a straightforward way.
8. Keep it concise under 25 words.

Format your response in JSON:
{{
    ""part"": ""<name of the specific part being pointed at>"",
    ""description"": ""<helpful explanation of that part in one sentence>""
}}
";
```

### Response Structure

```csharp
[Serializable]
private class PointingDescription
{
    public string part;        // e.g., "power button", "volume slider", "USB port"
    public string description; // e.g., "Press and hold for 3 seconds to turn on the device"
}
```

---

## Configuration Parameters

### SphereToggleScript.cs

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxHandProximityDistance` | 0.2f (20cm) | Max distance between wrists to enable pointing |
| `planeUpOffset` | 0.02f (2cm) | Vertical offset above fingertip for label |
| `inspectionUpdateInterval` | 5f | Seconds between Gemini API calls for part detection |
| `enableAutoRecordOnPointing` | true | Auto-start voice recording when pointing at new part |
| `autoRecordDuration` | 10f | Duration of auto-recording in seconds |
| `minTimeBetweenAutoRecordings` | 3f | Cooldown between auto-recordings |

### HandGrabTrigger.cs

| Parameter | Default | Description |
|-----------|---------|-------------|
| `confidenceThreshold` | 0.7f | Min Gemini confidence to trigger grab |
| `grabOffset` | Vector3.zero | Position offset when grabbed |
| `twinOffset` | (0.03f, -0.05f, 0f) | Twin object position relative to anchor |

---

## Event System

### Pointing State Events

```csharp
// Declared in SphereToggleScript.cs
public delegate void PointingStateChangedHandler(bool isPointing);
public static event PointingStateChangedHandler OnPointingStateChanged;

// Fired when pointing state changes
OnPointingStateChanged?.Invoke(currentlyPointing);
```

### Grab/Release Events

```csharp
// Declared in HandGrabTrigger.cs
public delegate void AnchorGrabEventHandler(SceneObjectAnchor anchor);
public static event AnchorGrabEventHandler OnAnchorGrabbed;
public static event AnchorGrabEventHandler OnAnchorReleased;
```

---

## XR Hand Joints Used

| Joint ID | Purpose |
|----------|---------|
| `XRHandJointID.Wrist` | Proximity detection between hands |
| `XRHandJointID.IndexTip` | Label positioning at fingertip |
| `XRHandJointID.IndexProximal` | Pointing gesture detection (angle calculation) |

---

## Visual Components

### pointingPlane GameObject

- **Type**: GameObject with TextMeshPro child
- **Purpose**: Displays part name and description at fingertip
- **Components**:
  - MeshRenderer (for plane visual)
  - TextMeshPro (pointingPlaneText for part name)
  - DualTargetLazyFollow (added at runtime for camera-facing rotation)

### Twin Object

- **Purpose**: Visual representation of grabbed object that follows hand
- **Spawned by**: ObjectMeshGenerator.EstimateAndGenerateObject()
- **Parented to**: Hand's parent transform
- **Destroyed**: When anchor is released

---

## State Machine

```
                    ┌─────────────────────┐
                    │   IDLE              │
                    │   (Not grabbed)     │
                    └─────────┬───────────┘
                              │ Grab detected
                              ▼
                    ┌─────────────────────┐
                    │   GRABBED           │
                    │   (Holding object)  │
                    └─────────┬───────────┘
                              │ Hands approach + pointing gesture
                              ▼
                    ┌─────────────────────┐
                    │   POINTING          │◄────────┐
                    │   (Detail inspect)  │         │
                    └─────────┬───────────┘         │
                              │                     │
              ┌───────────────┼───────────────┐     │
              │               │               │     │
              ▼               ▼               ▼     │
        Part Changed    Hands Separate   New Part  │
              │               │               └─────┘
              │               │
              ▼               ▼
        Generate Q's    Hide pointingPlane
        Update label    Return to GRABBED
```

---

## Integration with Other Systems

### Question Generation

When a new part is pointed at:
```csharp
StartCoroutine(GeneratePointingQuestionsRoutine(objectLabel, partName, partDescription));
```

### Voice Recording

Auto-recording triggers when pointing at new parts:
```csharp
if (isNewPart && enableAutoRecordOnPointing && !isAutoRecording)
{
    StartAutoRecording();
}
```

### User Study Logging

Events are logged for research purposes:
```csharp
LogUserStudy($"[DETAIL] [POINTING] POINTING_STARTED: Object=\"{labelUnderSphere.text}\", Part=\"{pointingPlaneText.text}\"");
```

---

## Debugging Tips

1. **Pointing not detected**: Check `maxHandProximityDistance` - hands might be too far apart
2. **Gesture not recognized**: Verify index finger angle threshold (25 degrees)
3. **Label not appearing**: Ensure `pointingPlane` is assigned in inspector
4. **Label in wrong position**: Check `planeUpOffset` value
5. **Label not facing camera**: Verify DualTargetLazyFollow is being added correctly

### Debug Logs to Watch

```
"Pointing state change - Hand distance: X.XXm, Index angle: XX.X, Is pointing: true/false"
"Updated pointing plane position for part: {partName}, at position: {position}"
"Pointing part changed from '{old}' to '{new}' - regenerating questions"
```
