# Programmable 3D Avatar Faces — Engines, APIs, and Real-Time Control for AI Assistants

You want your AI assistant to have a face. Not a static icon, not a pulsing circle — a real animated face that blinks, lip-syncs to speech, and shifts expression based on system state. Think VIKI from *I, Robot*, or a VTuber-style avatar that lives in your macOS menu bar. This article maps every viable path from "I want a programmable face" to "it's rendering at 60fps in a 50x60pt SwiftUI view."

**What you'll learn:**

- How the major face animation systems work (Live2D, SceneKit, Three.js, sprite sheets) and which ones you can actually control programmatically
- The ARKit blend shape standard and why it matters even when you're not doing face tracking
- How to map TTS output to mouth shapes in real-time (Rhubarb, OVR Visemes, Azure, Apple AVSpeechSynthesizer)
- Which cloud avatar APIs (HeyGen, Simli, D-ID) offer embeddable streaming and which are just video generators
- Why a 2D sprite-sheet approach might beat full 3D for a notch-sized display
- Production-ready code for every approach: SceneKit blend shapes, SpriteKit state machines, WKWebView + Three.js, and pure SwiftUI animation

---

## The Problem

Every AI assistant looks the same: a colored circle that pulses when it's "thinking." Siri has one. Alexa has one. ChatGPT has one. This is a missed opportunity. Humans are wired to read faces — micro-expressions, gaze direction, mouth movement. A face creates presence in a way that no amount of smooth gradient animation can match.

But building a programmable face is hard for several specific reasons:

**1. Most face engines assume face *tracking*, not face *generation*.** ARKit's blend shapes are designed to read your face via a TrueDepth camera, not to drive a synthetic face from code. You need to invert the pipeline.

**2. Game engines are overkill.** Unity and Unreal can absolutely render a talking face. But embedding a game engine runtime in a macOS notch HUD — a 50x60pt overlay window — is absurd. You need something lightweight.

**3. Lip sync is a separate problem from expression.** Getting the mouth to match speech requires phoneme-to-viseme mapping with sub-frame timing. Most "avatar" products handle this internally but don't expose the control surface.

**4. The display is tiny.** At 50x60 points (100x120 pixels on Retina), you don't need subsurface scattering or physically-based skin shading. You need *readable expression at low resolution*. This changes the entire technical calculus.

Here's the concrete scenario: you're building "Jane," an AI assistant that lives in a macOS notch-area HUD. The HUD is a borderless NSPanel that overlays the MacBook screen. Jane's face occupies roughly 50x60pt in the left ear of the notch. She needs to:

- Show 4-6 discrete expression states (calm, thinking, alert, alarmed, happy, speaking)
- Lip sync to TTS audio output
- Blink and perform idle animations (subtle head movement, gaze shifts)
- Transition smoothly between states
- Render at 60fps with negligible CPU/GPU impact
- Be driven entirely by code — no camera, no tracking, no user input

```swift
// The target API — what we want to be able to do
protocol AvatarController {
    func setExpression(_ expression: Expression, duration: TimeInterval)
    func setViseme(_ viseme: Viseme)
    func startIdleAnimation()
    func stopIdleAnimation()
    func blink()
    func setGazeDirection(_ direction: CGPoint) // -1...1 range
}

enum Expression: String, CaseIterable {
    case calm, thinking, alert, alarmed, happy, speaking, listening
}

enum Viseme: String, CaseIterable {
    case sil   // silence
    case pp    // p, b, m
    case ff    // f, v
    case th    // th
    case dd    // t, d
    case kk    // k, g
    case ch    // ch, j, sh
    case ss    // s, z
    case nn    // n, l
    case rr    // r
    case aa    // a
    case ee    // e
    case ih    // i
    case oh    // o
    case ou    // u
}
```

If you get this right, your assistant goes from a colored dot to a character with presence. Users will anthropomorphize it. They'll say "she looks worried" or "she's thinking about it." That's the goal.

---

## Core Concepts

### Blend Shapes (Morph Targets)

A blend shape is a named deformation of a base mesh. You have a neutral face mesh, and a set of target meshes that represent extremes — mouth fully open, left eyebrow fully raised, eyes fully closed. Each blend shape has a weight from 0.0 to 1.0. At runtime, you interpolate between the neutral mesh and the target meshes based on weights.

```typescript
// The blend shape concept as a type
interface BlendShapeState {
  // Mouth shapes (visemes)
  jawOpen: number;        // 0.0 - 1.0
  mouthFunnel: number;
  mouthPucker: number;
  mouthLeft: number;
  mouthRight: number;
  mouthSmileLeft: number;
  mouthSmileRight: number;
  mouthFrownLeft: number;
  mouthFrownRight: number;

  // Eyes
  eyeBlinkLeft: number;
  eyeBlinkRight: number;
  eyeLookUpLeft: number;
  eyeLookDownLeft: number;
  eyeLookInLeft: number;
  eyeLookOutLeft: number;

  // Brows
  browDownLeft: number;
  browDownRight: number;
  browInnerUp: number;
  browOuterUpLeft: number;
  browOuterUpRight: number;
}

// An expression is just a preset blend shape configuration
const EXPRESSIONS: Record<string, Partial<BlendShapeState>> = {
  calm: {
    mouthSmileLeft: 0.1,
    mouthSmileRight: 0.1,
    eyeBlinkLeft: 0.05,
    eyeBlinkRight: 0.05,
  },
  thinking: {
    browInnerUp: 0.3,
    eyeLookUpLeft: 0.2,
    eyeLookUpRight: 0.2,
    mouthPucker: 0.15,
  },
  alarmed: {
    eyeBlinkLeft: 0.0,
    eyeBlinkRight: 0.0,
    browInnerUp: 0.7,
    browOuterUpLeft: 0.5,
    browOuterUpRight: 0.5,
    jawOpen: 0.2,
    mouthFrownLeft: 0.3,
    mouthFrownRight: 0.3,
  },
};
```

> **Key insight:** ARKit defines 52 blend shape locations. This has become the de facto standard — not because ARKit is the only system, but because every major avatar platform (Ready Player Me, VRChat, MetaHuman) supports ARKit-compatible blend shapes. If your face model uses ARKit naming, it will work with virtually any animation pipeline.

The ARKit blend shape set is documented at [ARFaceAnchor.BlendShapeLocation](https://developer.apple.com/documentation/arkit/arfaceanchor/blendshapelocation). The critical thing to understand is that these values are *just numbers*. You don't need a TrueDepth camera to use them. You can set `jawOpen = 0.6` from code and render the result in SceneKit.

### Visemes

A viseme is the visual representation of a phoneme — the mouth shape that corresponds to a speech sound. The most widely used standard is the [Oculus OVR LipSync set](https://developers.meta.com/horizon/documentation/unity/audio-ovrlipsync-viseme-reference/) of 15 visemes:

```typescript
// The OVR viseme set — 15 mouth shapes cover all English speech
interface VisemeSet {
  sil: BlendShapeState;  // Silence / neutral
  PP: BlendShapeState;   // p, b, m — lips pressed together
  FF: BlendShapeState;   // f, v — lower lip to upper teeth
  TH: BlendShapeState;   // th — tongue between teeth
  DD: BlendShapeState;   // t, d — tongue to palate
  kk: BlendShapeState;   // k, g — back of tongue up
  CH: BlendShapeState;   // ch, j, sh — wide mouth
  SS: BlendShapeState;   // s, z — teeth together, lips slightly apart
  nn: BlendShapeState;   // n, l — tongue to palate, mouth relaxed
  RR: BlendShapeState;   // r — lips slightly rounded
  aa: BlendShapeState;   // a — wide open
  E: BlendShapeState;    // e — wide, mid-height
  ih: BlendShapeState;   // i — narrow, high
  oh: BlendShapeState;   // o — rounded, mid-height
  ou: BlendShapeState;   // u — rounded, small
}

// Mapping visemes to ARKit blend shapes
const VISEME_TO_BLENDSHAPE: Record<string, Partial<BlendShapeState>> = {
  sil: { jawOpen: 0.0, mouthFunnel: 0.0, mouthPucker: 0.0 },
  PP:  { jawOpen: 0.0, mouthPucker: 0.3, mouthFunnel: 0.0 },
  FF:  { jawOpen: 0.05, mouthFunnel: 0.2, mouthSmileLeft: 0.1 },
  TH:  { jawOpen: 0.1, mouthFunnel: 0.1 },
  DD:  { jawOpen: 0.15, mouthSmileLeft: 0.05, mouthSmileRight: 0.05 },
  kk:  { jawOpen: 0.2, mouthFunnel: 0.1 },
  CH:  { jawOpen: 0.15, mouthSmileLeft: 0.2, mouthSmileRight: 0.2 },
  SS:  { jawOpen: 0.05, mouthSmileLeft: 0.15, mouthSmileRight: 0.15 },
  nn:  { jawOpen: 0.1, mouthSmileLeft: 0.05, mouthSmileRight: 0.05 },
  RR:  { jawOpen: 0.15, mouthPucker: 0.2, mouthFunnel: 0.15 },
  aa:  { jawOpen: 0.5, mouthSmileLeft: 0.0, mouthSmileRight: 0.0 },
  E:   { jawOpen: 0.3, mouthSmileLeft: 0.2, mouthSmileRight: 0.2 },
  ih:  { jawOpen: 0.15, mouthSmileLeft: 0.3, mouthSmileRight: 0.3 },
  oh:  { jawOpen: 0.35, mouthPucker: 0.15, mouthFunnel: 0.25 },
  ou:  { jawOpen: 0.2, mouthPucker: 0.35, mouthFunnel: 0.3 },
};
```

> **Key insight:** 15 visemes is the sweet spot. Fewer and the mouth looks robotic. More and you can't interpolate smoothly at real-time speeds. The OVR set is language-agnostic — the same 15 shapes cover English, Japanese, French, and most other languages because visemes map to *mouth shapes*, not linguistic units.

### Expression State Machine

Expressions aren't instantaneous — you transition between them. A state machine manages this:

```typescript
interface ExpressionTransition {
  from: string;
  to: string;
  duration: number;        // seconds
  easing: EasingFunction;
  blendShapeDeltas: Partial<BlendShapeState>;
}

type EasingFunction = 'linear' | 'easeIn' | 'easeOut' | 'easeInOut' | 'spring';

// Expression layers — allow lip sync and expression to coexist
interface AnimationLayer {
  name: string;
  weight: number;
  blendShapes: Partial<BlendShapeState>;
  priority: number; // higher priority layers override lower ones
}

// Layer system: expression + lip sync + idle can all run simultaneously
// Layer 0: Idle animation (blinks, micro-movements)
// Layer 1: Expression state (calm, thinking, alarmed)
// Layer 2: Lip sync (viseme animation — overrides mouth-related shapes only)
// Layer 3: Overrides (forced blink, gaze direction)

function composeLayers(layers: AnimationLayer[]): BlendShapeState {
  const sorted = layers.sort((a, b) => a.priority - b.priority);
  const result: Partial<BlendShapeState> = {};

  for (const layer of sorted) {
    for (const [key, value] of Object.entries(layer.blendShapes)) {
      const existing = result[key as keyof BlendShapeState] ?? 0;
      // Blend based on layer weight
      result[key as keyof BlendShapeState] =
        existing * (1 - layer.weight) + (value as number) * layer.weight;
    }
  }

  return result as BlendShapeState;
}
```

> **Key insight:** Never apply lip sync and expression on the same blend shapes without a layering system. If "alarmed" sets `jawOpen: 0.2` and the current viseme sets `jawOpen: 0.5`, you need a rule for resolving the conflict. The standard approach is to let lip sync override mouth shapes while expression controls brows and eyes.

---

## Architecture Decision: 3D vs 2D vs Hybrid

Before diving into specific engines, you need to make a fundamental choice. At 50x60pt (100x120 pixels on Retina), do you even need 3D?

### Option A: Full 3D (SceneKit / RealityKit / Metal)

A 3D face mesh with blend shapes, lit and rendered in real-time.

**Pros:**
- Smooth interpolation between any expression
- Continuous viseme blending (not discrete frames)
- Head rotation, lighting changes, particle effects
- Looks sophisticated even at small sizes — 3D depth cues read well

**Cons:**
- Requires a 3D model with properly weighted blend shapes
- GPU overhead (small but nonzero)
- More complex pipeline — model authoring, rigging, shader setup

### Option B: 2D Sprite Sheets (SpriteKit / SwiftUI Animation)

Pre-rendered frames for each expression and viseme, played as sprite animations.

**Pros:**
- Zero GPU overhead — it's just image swapping
- Perfect quality at target size (pre-rendered at exact resolution)
- Dead simple implementation
- Artist-friendly — any illustrator can make the frames

**Cons:**
- Combinatorial explosion: 7 expressions x 15 visemes x 3 blink states = 315 frames minimum
- Transitions between expressions are stepped, not smooth
- No continuous head rotation or gaze direction
- Adding a new expression means drawing more frames

### Option C: Hybrid (2D layers with programmatic composition)

Separate layers for face base, eyes, mouth, brows — each independently animated.

**Pros:**
- Reduces combinatorial explosion (separate layers multiply, don't explode)
- Smooth animation via layer-level transforms
- Good quality at small sizes
- Moderate complexity

**Cons:**
- Parallax and depth cues are limited
- Layer registration requires precision
- Still need pre-rendered assets per layer

### Option D: WebView-based (WKWebView + Three.js / TalkingHead)

A web-based renderer embedded in a native macOS app via WKWebView.

**Pros:**
- Enormous ecosystem of Three.js face animation tools
- Ready Player Me avatars work out of the box
- TalkingHead library provides complete lip sync pipeline
- Cross-platform compatible

**Cons:**
- WKWebView has overhead (separate process, IPC)
- WebGL performance in WKWebView is inconsistent on macOS
- Bridge between Swift and JavaScript adds latency
- Memory footprint is much larger than native approaches

### Recommendation for the Jane Use Case

For a 50x60pt notch display, **Option C (Hybrid 2D layers) is the best starting point**, with **Option A (SceneKit)** as the upgrade path. Here's why:

At this size, 3D depth cues are barely perceptible. What matters is *readable expression*. A well-designed set of 2D layers — stylized, high-contrast, designed for the target resolution — will read better than a realistic 3D face shrunk to 100x120 pixels. The VIKI aesthetic (translucent, holographic, stylized) actually plays to 2D's strengths.

If you later want the avatar to appear in a larger context (a floating panel, a full-screen takeover), that's when you upgrade to SceneKit with blend shapes.

---

## Pattern 1: SceneKit Blend Shape Avatar

This is the gold standard for native macOS 3D face rendering. SceneKit is Apple's high-level 3D framework, built on Metal, with first-class SwiftUI integration.

### Setup: Loading a Face Model with Morph Targets

You need a 3D model (GLTF/USDZ/DAE) with blend shape targets. The model should have ARKit-compatible blend shape names. You can create one in Blender using the [ARKit Blendshape Helper](https://github.com/elijah-atkins/ARKitBlendshapeHelper) addon, or use a Ready Player Me avatar exported as GLB.

```swift
import SceneKit
import SwiftUI

class FaceSceneController: ObservableObject {
    let scene = SCNScene()
    private var faceNode: SCNNode?
    private var morpher: SCNMorpher?

    // Blend shape target indices — mapped from ARKit names
    private var blendShapeIndices: [String: Int] = [:]

    // Animation layers
    private var expressionLayer: [String: CGFloat] = [:]
    private var visemeLayer: [String: CGFloat] = [:]
    private var idleLayer: [String: CGFloat] = [:]

    // Idle animation state
    private var idleTimer: Timer?
    private var blinkTimer: Timer?
    private var nextBlinkTime: TimeInterval = 0

    func loadFaceModel(named modelName: String) {
        guard let modelScene = SCNScene(named: modelName) else {
            fatalError("Could not load face model: \(modelName)")
        }

        // Find the face mesh node (assumes it's named "Face" or "Head")
        guard let face = modelScene.rootNode.childNode(
            withName: "Face", recursively: true
        ) else {
            fatalError("No 'Face' node found in model")
        }

        faceNode = face
        scene.rootNode.addChildNode(face)

        // Access the morpher (blend shape controller)
        guard let m = face.morpher else {
            fatalError("Face node has no morpher — ensure model has blend shapes")
        }
        morpher = m
        m.calculationMode = .additive

        // Build index map from target names
        for (index, target) in m.targets.enumerated() {
            if let name = target.name {
                blendShapeIndices[name] = index
            }
        }

        setupCamera()
        setupLighting()
    }

    private func setupCamera() {
        let camera = SCNCamera()
        camera.fieldOfView = 30
        camera.zNear = 0.1
        camera.zFar = 100

        let cameraNode = SCNNode()
        cameraNode.camera = camera
        cameraNode.position = SCNVector3(0, 0, 0.5) // Close-up on face
        cameraNode.look(at: SCNVector3Zero)
        scene.rootNode.addChildNode(cameraNode)
    }

    private func setupLighting() {
        // Soft ambient + directional for readable features at small sizes
        let ambient = SCNLight()
        ambient.type = .ambient
        ambient.color = NSColor(white: 0.4, alpha: 1)
        let ambientNode = SCNNode()
        ambientNode.light = ambient
        scene.rootNode.addChildNode(ambientNode)

        let directional = SCNLight()
        directional.type = .directional
        directional.color = NSColor(white: 0.8, alpha: 1)
        directional.castsShadow = false
        let dirNode = SCNNode()
        dirNode.light = directional
        dirNode.eulerAngles = SCNVector3(-Float.pi / 4, Float.pi / 6, 0)
        scene.rootNode.addChildNode(dirNode)
    }

    // MARK: - Blend Shape Control

    func setBlendShape(_ name: String, weight: CGFloat, animated: Bool = true) {
        guard let index = blendShapeIndices[name] else { return }

        if animated {
            SCNTransaction.begin()
            SCNTransaction.animationDuration = 0.1
            morpher?.setWeight(weight, forTargetAt: index)
            SCNTransaction.commit()
        } else {
            morpher?.setWeight(weight, forTargetAt: index)
        }
    }

    func setExpression(_ expression: Expression, duration: TimeInterval = 0.3) {
        let targets = expression.blendShapeTargets

        SCNTransaction.begin()
        SCNTransaction.animationDuration = duration
        SCNTransaction.animationTimingFunction =
            CAMediaTimingFunction(name: .easeInEaseOut)

        // Reset all expression-controlled shapes
        for name in Expression.expressionShapeNames {
            if let index = blendShapeIndices[name] {
                let visemeValue = visemeLayer[name] ?? 0
                let targetValue = targets[name] ?? 0
                // Expression sets base, viseme overrides mouth shapes
                if Expression.mouthShapeNames.contains(name) {
                    morpher?.setWeight(max(targetValue, visemeValue), forTargetAt: index)
                } else {
                    morpher?.setWeight(targetValue, forTargetAt: index)
                }
            }
        }

        SCNTransaction.commit()
        expressionLayer = targets
    }

    func setViseme(_ viseme: Viseme, weight: CGFloat = 1.0) {
        let targets = viseme.blendShapeTargets

        // Visemes update immediately — no animation duration
        // The calling code handles interpolation timing
        for (name, value) in targets {
            guard let index = blendShapeIndices[name] else { continue }
            let scaledValue = value * weight
            morpher?.setWeight(scaledValue, forTargetAt: index)
            visemeLayer[name] = scaledValue
        }
    }

    // MARK: - Idle Animation

    func startIdleAnimation() {
        // Blink timer — humans blink every 2-10 seconds
        blinkTimer = Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) {
            [weak self] _ in
            self?.updateBlink()
        }
        scheduleNextBlink()

        // Micro-movement timer — subtle head sway
        idleTimer = Timer.scheduledTimer(withTimeInterval: 1.0 / 30.0, repeats: true) {
            [weak self] _ in
            self?.updateIdleMovement()
        }
    }

    func stopIdleAnimation() {
        blinkTimer?.invalidate()
        idleTimer?.invalidate()
        blinkTimer = nil
        idleTimer = nil
    }

    private func scheduleNextBlink() {
        nextBlinkTime = CACurrentMediaTime() + Double.random(in: 2.0...6.0)
    }

    private func updateBlink() {
        let now = CACurrentMediaTime()
        guard now >= nextBlinkTime else { return }

        // Blink animation: close over 0.05s, hold 0.05s, open over 0.1s
        SCNTransaction.begin()
        SCNTransaction.animationDuration = 0.05
        setBlendShape("eyeBlinkLeft", weight: 1.0, animated: false)
        setBlendShape("eyeBlinkRight", weight: 1.0, animated: false)
        SCNTransaction.commit()

        DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) { [weak self] in
            SCNTransaction.begin()
            SCNTransaction.animationDuration = 0.1
            self?.setBlendShape("eyeBlinkLeft", weight: 0.0, animated: false)
            self?.setBlendShape("eyeBlinkRight", weight: 0.0, animated: false)
            SCNTransaction.commit()
        }

        scheduleNextBlink()
    }

    private func updateIdleMovement() {
        let time = CACurrentMediaTime()
        // Perlin-noise-like subtle head movement
        let yaw = sin(time * 0.3) * 0.02
        let pitch = sin(time * 0.5 + 1.0) * 0.01
        faceNode?.eulerAngles.y = Float(yaw)
        faceNode?.eulerAngles.x = Float(pitch)
    }

    func setGazeDirection(_ direction: CGPoint) {
        // Map -1...1 to blend shape weights
        let lookRight = max(0, direction.x)
        let lookLeft = max(0, -direction.x)
        let lookUp = max(0, direction.y)
        let lookDown = max(0, -direction.y)

        setBlendShape("eyeLookOutLeft", weight: lookLeft * 0.5)
        setBlendShape("eyeLookInRight", weight: lookLeft * 0.5)
        setBlendShape("eyeLookInLeft", weight: lookRight * 0.5)
        setBlendShape("eyeLookOutRight", weight: lookRight * 0.5)
        setBlendShape("eyeLookUpLeft", weight: lookUp * 0.3)
        setBlendShape("eyeLookUpRight", weight: lookUp * 0.3)
        setBlendShape("eyeLookDownLeft", weight: lookDown * 0.3)
        setBlendShape("eyeLookDownRight", weight: lookDown * 0.3)
    }
}
```

### Embedding in SwiftUI

```swift
import SwiftUI
import SceneKit

struct AvatarSceneView: NSViewRepresentable {
    @ObservedObject var controller: FaceSceneController

    func makeNSView(context: Context) -> SCNView {
        let scnView = SCNView()
        scnView.scene = controller.scene
        scnView.backgroundColor = .clear
        scnView.allowsCameraControl = false
        scnView.preferredFramesPerSecond = 60
        scnView.antialiasingMode = .multisampling4X
        scnView.isJitteringEnabled = false

        // For small views, reduce rendering quality for performance
        scnView.antialiasingMode = .none // At 50x60pt, AA is invisible
        scnView.preferredFramesPerSecond = 30 // 30fps is sufficient for expressions

        return scnView
    }

    func updateNSView(_ nsView: SCNView, context: Context) {
        // Scene updates happen through the controller, not through SwiftUI diffing
    }
}

// Usage in the HUD
struct AvatarZone: View {
    @StateObject private var faceController = FaceSceneController()

    var body: some View {
        AvatarSceneView(controller: faceController)
            .frame(maxWidth: .infinity, maxHeight: .infinity)
            .clipShape(UnevenRoundedRectangle(
                topLeadingRadius: 12,
                bottomLeadingRadius: 12,
                bottomTrailingRadius: 0,
                topTrailingRadius: 0
            ))
            .onAppear {
                faceController.loadFaceModel(named: "jane_face.usdz")
                faceController.startIdleAnimation()
                faceController.setExpression(.calm)
            }
    }
}
```

> **Key insight:** At 50x60pt, turn off antialiasing (`antialiasingMode = .none`) and drop to 30fps. The visual difference is imperceptible at this size, but the GPU savings are real. You can always render at higher quality when the avatar appears in a larger context.

---

## Pattern 2: SpriteKit State Machine Avatar

For the absolute lightest-weight approach, use pre-rendered sprite frames driven by a state machine. This is ideal for the Jane notch HUD.

```swift
import SpriteKit
import SwiftUI

// MARK: - Avatar Sprite Sheet Definition

struct AvatarSpriteSheet {
    // Each expression has a set of mouth frames (one per viseme)
    // and eye states (open, half, closed)
    struct ExpressionSet {
        let baseFace: String           // Asset name for the base face
        let visemeFrames: [Viseme: String] // Asset name per viseme
        let eyeOpen: String
        let eyeHalf: String
        let eyeClosed: String
        let browTexture: String        // Expression-specific brow
    }

    static let expressions: [Expression: ExpressionSet] = [
        .calm: ExpressionSet(
            baseFace: "face_calm",
            visemeFrames: Viseme.allCases.reduce(into: [:]) { $0[$1] = "mouth_calm_\($1.rawValue)" },
            eyeOpen: "eyes_calm_open",
            eyeHalf: "eyes_calm_half",
            eyeClosed: "eyes_calm_closed",
            browTexture: "brow_calm"
        ),
        .thinking: ExpressionSet(
            baseFace: "face_thinking",
            visemeFrames: Viseme.allCases.reduce(into: [:]) { $0[$1] = "mouth_thinking_\($1.rawValue)" },
            eyeOpen: "eyes_thinking_open",
            eyeHalf: "eyes_thinking_half",
            eyeClosed: "eyes_thinking_closed",
            browTexture: "brow_thinking"
        ),
        .alarmed: ExpressionSet(
            baseFace: "face_alarmed",
            visemeFrames: Viseme.allCases.reduce(into: [:]) { $0[$1] = "mouth_alarmed_\($1.rawValue)" },
            eyeOpen: "eyes_alarmed_open",
            eyeHalf: "eyes_alarmed_half",
            eyeClosed: "eyes_alarmed_closed",
            browTexture: "brow_alarmed"
        ),
        // ... more expressions
    ]
}

// MARK: - SpriteKit Avatar Scene

class AvatarSpriteScene: SKScene {

    // Sprite layers (back to front)
    private let faceSprite = SKSpriteNode()
    private let mouthSprite = SKSpriteNode()
    private let eyesSprite = SKSpriteNode()
    private let browSprite = SKSpriteNode()
    private let overlaySprite = SKSpriteNode() // holographic effects

    // State
    private var currentExpression: Expression = .calm
    private var currentViseme: Viseme = .sil
    private var isBlinking = false
    private var nextBlinkTime: TimeInterval = 0
    private var gazeOffset: CGPoint = .zero

    override func didMove(to view: SKView) {
        backgroundColor = .clear
        scaleMode = .aspectFill

        // Layer setup — all centered
        let layers: [SKSpriteNode] = [faceSprite, mouthSprite, eyesSprite, browSprite, overlaySprite]
        for (index, layer) in layers.enumerated() {
            layer.position = CGPoint(x: size.width / 2, y: size.height / 2)
            layer.zPosition = CGFloat(index)
            layer.size = size
            addChild(layer)
        }

        setExpression(.calm)
        scheduleNextBlink()
    }

    // MARK: - Expression Control

    func setExpression(_ expression: Expression, animated: Bool = true) {
        guard let set = AvatarSpriteSheet.expressions[expression] else { return }
        currentExpression = expression

        let update = {
            self.faceSprite.texture = SKTexture(imageNamed: set.baseFace)
            self.eyesSprite.texture = SKTexture(imageNamed: set.eyeOpen)
            self.browSprite.texture = SKTexture(imageNamed: set.browTexture)
            self.updateMouthForCurrentViseme()
        }

        if animated {
            // Cross-fade transition
            let fadeOut = SKAction.fadeAlpha(to: 0.7, duration: 0.1)
            let changeTextures = SKAction.run(update)
            let fadeIn = SKAction.fadeAlpha(to: 1.0, duration: 0.15)
            let sequence = SKAction.sequence([fadeOut, changeTextures, fadeIn])

            faceSprite.run(sequence)
            eyesSprite.run(sequence)
            browSprite.run(sequence)
        } else {
            update()
        }
    }

    // MARK: - Viseme Control

    func setViseme(_ viseme: Viseme) {
        currentViseme = viseme
        updateMouthForCurrentViseme()
    }

    private func updateMouthForCurrentViseme() {
        guard let set = AvatarSpriteSheet.expressions[currentExpression] else { return }
        if let textureName = set.visemeFrames[currentViseme] {
            mouthSprite.texture = SKTexture(imageNamed: textureName)
        }
    }

    // MARK: - Blink System

    private func scheduleNextBlink() {
        nextBlinkTime = CACurrentMediaTime() + Double.random(in: 2.5...5.5)
    }

    override func update(_ currentTime: TimeInterval) {
        // Blink check
        if !isBlinking && CACurrentMediaTime() >= nextBlinkTime {
            performBlink()
        }

        // Subtle eye position from gaze
        eyesSprite.position = CGPoint(
            x: size.width / 2 + gazeOffset.x * 2,  // 2pt max eye shift
            y: size.height / 2 + gazeOffset.y * 1.5
        )
    }

    private func performBlink() {
        isBlinking = true
        guard let set = AvatarSpriteSheet.expressions[currentExpression] else { return }

        let halfClose = SKAction.run {
            self.eyesSprite.texture = SKTexture(imageNamed: set.eyeHalf)
        }
        let fullClose = SKAction.run {
            self.eyesSprite.texture = SKTexture(imageNamed: set.eyeClosed)
        }
        let fullOpen = SKAction.run {
            self.eyesSprite.texture = SKTexture(imageNamed: set.eyeOpen)
            self.isBlinking = false
            self.scheduleNextBlink()
        }

        let sequence = SKAction.sequence([
            halfClose,
            SKAction.wait(forDuration: 0.03),
            fullClose,
            SKAction.wait(forDuration: 0.06),
            halfClose,
            SKAction.wait(forDuration: 0.03),
            fullOpen,
        ])

        eyesSprite.run(sequence)
    }

    // MARK: - Gaze

    func setGazeDirection(_ direction: CGPoint) {
        gazeOffset = direction
    }
}

// MARK: - SwiftUI Integration

struct SpriteAvatarView: View {
    let scene: AvatarSpriteScene

    init() {
        let s = AvatarSpriteScene(size: CGSize(width: 100, height: 120))
        s.scaleMode = .aspectFill
        scene = s
    }

    var body: some View {
        SpriteView(scene: scene, options: [.allowsTransparency])
            .frame(maxWidth: .infinity, maxHeight: .infinity)
    }
}
```

**Asset production for the sprite approach:**

For a layered 2D avatar, you need these assets per expression:

| Layer | Assets needed | Per expression |
|-------|--------------|----------------|
| Base face | 1 texture | x 7 expressions = 7 |
| Mouth | 15 viseme textures | x 7 = 105 |
| Eyes | 3 states (open/half/closed) | x 7 = 21 |
| Brows | 1 texture | x 7 = 7 |
| **Total** | | **140 textures** |

At 100x120px (Retina), each PNG is about 5-15KB. Total asset size: ~1-2MB. Trivial.

> **Key insight:** The layered sprite approach separates the combinatorial problem. Instead of 7 x 15 x 3 = 315 pre-composed frames, you have 140 layer textures that combine at runtime. The SpriteKit compositor handles the layering with zero effort.

---

## Pattern 3: WKWebView + TalkingHead (Three.js)

If you want a full 3D avatar with minimal native code, embed a web-based renderer. The [TalkingHead](https://github.com/met4citizen/TalkingHead) library is the most complete open-source solution — it handles Ready Player Me avatars, lip sync, expressions, and idle animations out of the box.

```swift
import SwiftUI
import WebKit

class AvatarWebViewController: NSObject, ObservableObject, WKScriptMessageHandler {
    let webView: WKWebView

    init(frame: CGRect = .zero) {
        let config = WKWebViewConfiguration()
        config.preferences.setValue(true, forKey: "developerExtrasEnabled")

        // Enable WebGL
        let prefs = WKWebpagePreferences()
        prefs.allowsContentJavaScript = true
        config.defaultWebpagePreferences = prefs

        let contentController = WKUserContentController()
        config.userContentController = contentController

        webView = WKWebView(frame: frame, configuration: config)
        webView.setValue(false, forKey: "drawsBackground") // Transparent background

        super.init()

        contentController.add(self, name: "avatarBridge")
        loadAvatarPage()
    }

    private func loadAvatarPage() {
        let html = Self.avatarHTML
        webView.loadHTMLString(html, baseURL: Bundle.main.resourceURL)
    }

    // MARK: - Control API

    func setExpression(_ expression: String, duration: Double = 0.3) {
        callJS("setExpression('\(expression)', \(duration))")
    }

    func setViseme(_ viseme: String, weight: Double = 1.0) {
        callJS("setViseme('\(viseme)', \(weight))")
    }

    func speak(_ text: String) {
        let escaped = text.replacingOccurrences(of: "'", with: "\\'")
        callJS("speak('\(escaped)')")
    }

    func setMood(_ mood: String, intensity: Double = 0.5) {
        callJS("setMood('\(mood)', \(intensity))")
    }

    private func callJS(_ script: String) {
        webView.evaluateJavaScript(script) { _, error in
            if let error = error {
                print("JS error: \(error.localizedDescription)")
            }
        }
    }

    // MARK: - WKScriptMessageHandler

    func userContentController(
        _ userContentController: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        // Receive events from the web avatar (e.g., animation complete, error)
        guard let body = message.body as? [String: Any] else { return }
        if let event = body["event"] as? String {
            print("Avatar event: \(event)")
        }
    }

    // MARK: - HTML Template

    static let avatarHTML = """
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="utf-8">
        <style>
            * { margin: 0; padding: 0; }
            body { background: transparent; overflow: hidden; }
            canvas { width: 100%; height: 100%; }
        </style>
    </head>
    <body>
        <script type="importmap">
        {
            "imports": {
                "three": "https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.module.js",
                "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.170.0/examples/jsm/",
                "talkinghead": "https://cdn.jsdelivr.net/npm/talkinghead@latest/modules/talkinghead.mjs"
            }
        }
        </script>
        <script type="module">
            import { TalkingHead } from 'talkinghead';

            let head;

            async function init() {
                const container = document.body;
                head = new TalkingHead(container, {
                    ttsEndpoint: null, // We'll drive TTS from Swift
                    cameraView: 'head',
                    cameraDistance: 0.4,
                    cameraRotateX: 0,
                    cameraRotateY: 0,
                    lightAmbientColor: 0x404040,
                    lightDirectColor: 0xcccccc,
                    background: { color: 'transparent' },
                });

                // Load a Ready Player Me avatar
                await head.showAvatar({
                    url: 'https://models.readyplayer.me/64bfa15f0e72c63d7c3934a6.glb?morphTargets=ARKit&textureAtlas=1024',
                    body: 'F',
                    avatarMood: 'neutral',
                    lipsyncLang: 'en',
                });

                // Notify Swift that avatar is ready
                window.webkit.messageHandlers.avatarBridge.postMessage({
                    event: 'ready'
                });
            }

            // API exposed to Swift via evaluateJavaScript
            window.setExpression = (name, duration) => {
                if (!head) return;
                const moods = {
                    calm: { mood: 'neutral', intensity: 0.3 },
                    thinking: { mood: 'concern', intensity: 0.4 },
                    alert: { mood: 'surprise', intensity: 0.5 },
                    alarmed: { mood: 'anger', intensity: 0.6 },
                    happy: { mood: 'joy', intensity: 0.7 },
                    listening: { mood: 'interest', intensity: 0.4 },
                };
                const m = moods[name] || moods.calm;
                head.setMood(m.mood);
            };

            window.setViseme = (viseme, weight) => {
                if (!head) return;
                // TalkingHead handles viseme interpolation internally
                // For manual control, directly set morph targets
                head.setMorphTarget(viseme, weight);
            };

            window.speak = (text) => {
                if (!head) return;
                head.speakText(text);
            };

            window.setMood = (mood, intensity) => {
                if (!head) return;
                head.setMood(mood);
            };

            init().catch(console.error);
        </script>
    </body>
    </html>
    """;
}

// MARK: - SwiftUI Wrapper

struct WebAvatarView: NSViewRepresentable {
    @ObservedObject var controller: AvatarWebViewController

    func makeNSView(context: Context) -> WKWebView {
        controller.webView
    }

    func updateNSView(_ nsView: WKWebView, context: Context) {}
}
```

> **Key insight:** The WKWebView approach trades performance for ecosystem access. TalkingHead gives you Ready Player Me compatibility, built-in lip sync for multiple languages, emotion-to-blend-shape mapping, and Mixamo animation support — all for free. The cost is ~50MB of memory overhead and potential WebGL rendering inconsistencies. For a notch HUD, this may be too heavy. For a floating panel or full-screen mode, it's excellent.

---

## Pattern 4: Pure SwiftUI Animated Avatar

For the absolute simplest approach — no SpriteKit, no SceneKit, no WebView — you can build a face entirely in SwiftUI with shapes and animations.

```swift
import SwiftUI

// MARK: - Stylized Face Components

struct StylizedEye: View {
    let openness: CGFloat   // 0 = closed, 1 = fully open
    let lookDirection: CGPoint // -1...1

    var body: some View {
        ZStack {
            // Eye socket
            Ellipse()
                .fill(.white.opacity(0.9))
                .frame(width: 14, height: 10 * openness)
                .animation(.easeInOut(duration: 0.05), value: openness)

            // Iris
            Circle()
                .fill(
                    RadialGradient(
                        colors: [.cyan, .blue.opacity(0.8)],
                        center: .center,
                        startRadius: 0,
                        endRadius: 4
                    )
                )
                .frame(width: 6, height: 6)
                .offset(
                    x: lookDirection.x * 2,
                    y: lookDirection.y * 1.5
                )
                .opacity(openness > 0.3 ? 1 : 0)
                .animation(.easeOut(duration: 0.1), value: lookDirection)

            // Pupil
            Circle()
                .fill(.black)
                .frame(width: 3, height: 3)
                .offset(
                    x: lookDirection.x * 2,
                    y: lookDirection.y * 1.5
                )
                .opacity(openness > 0.3 ? 1 : 0)
        }
    }
}

struct StylizedMouth: View {
    let viseme: Viseme
    let expressionSmile: CGFloat // -1 (frown) to 1 (smile)

    private var mouthShape: some Shape {
        MouthPath(viseme: viseme, smile: expressionSmile)
    }

    var body: some View {
        mouthShape
            .fill(.white.opacity(0.7))
            .frame(width: 16, height: 10)
            .animation(.easeOut(duration: 0.05), value: viseme)
            .animation(.easeInOut(duration: 0.2), value: expressionSmile)
    }
}

struct MouthPath: Shape {
    var viseme: Viseme
    var smile: CGFloat

    var animatableData: CGFloat {
        get { smile }
        set { smile = newValue }
    }

    func path(in rect: CGRect) -> Path {
        var path = Path()
        let w = rect.width
        let h = rect.height
        let cx = rect.midX
        let cy = rect.midY

        switch viseme {
        case .sil:
            // Closed mouth — slight line
            let smileY = -smile * h * 0.1
            path.move(to: CGPoint(x: cx - w * 0.3, y: cy + smileY))
            path.addQuadCurve(
                to: CGPoint(x: cx + w * 0.3, y: cy + smileY),
                control: CGPoint(x: cx, y: cy - smileY * 2)
            )
        case .aa:
            // Wide open mouth
            path.addEllipse(in: CGRect(
                x: cx - w * 0.25, y: cy - h * 0.3,
                width: w * 0.5, height: h * 0.6
            ))
        case .oh:
            // Rounded O shape
            path.addEllipse(in: CGRect(
                x: cx - w * 0.15, y: cy - h * 0.25,
                width: w * 0.3, height: h * 0.5
            ))
        case .ee:
            // Wide smile
            path.addEllipse(in: CGRect(
                x: cx - w * 0.3, y: cy - h * 0.1,
                width: w * 0.6, height: h * 0.25
            ))
        case .pp:
            // Lips pressed together
            path.move(to: CGPoint(x: cx - w * 0.2, y: cy))
            path.addLine(to: CGPoint(x: cx + w * 0.2, y: cy))
        default:
            // Generic slightly open mouth for other visemes
            path.addEllipse(in: CGRect(
                x: cx - w * 0.2, y: cy - h * 0.15,
                width: w * 0.4, height: h * 0.3
            ))
        }

        return path
    }
}

// MARK: - Holographic Face Composite

struct HolographicFace: View {
    @StateObject private var animator = FaceAnimator()

    var body: some View {
        ZStack {
            // Scanline overlay (VIKI aesthetic)
            ScanlineOverlay()
                .opacity(0.15)

            // Face outline — holographic glow
            FaceOutline()
                .stroke(
                    LinearGradient(
                        colors: [.cyan.opacity(0.6), .blue.opacity(0.3)],
                        startPoint: .top,
                        endPoint: .bottom
                    ),
                    lineWidth: 1
                )

            // Eyes
            HStack(spacing: 10) {
                StylizedEye(
                    openness: animator.eyeOpenness,
                    lookDirection: animator.gazeDirection
                )
                StylizedEye(
                    openness: animator.eyeOpenness,
                    lookDirection: animator.gazeDirection
                )
            }
            .offset(y: -8)

            // Brows
            HStack(spacing: 14) {
                BrowShape(raise: animator.browRaise, furrow: animator.browFurrow)
                    .rotation3DEffect(.degrees(180), axis: (x: 0, y: 1, z: 0))
                BrowShape(raise: animator.browRaise, furrow: animator.browFurrow)
            }
            .offset(y: -16)

            // Mouth
            StylizedMouth(
                viseme: animator.currentViseme,
                expressionSmile: animator.mouthSmile
            )
            .offset(y: 10)
        }
        .frame(width: 50, height: 60)
        .background(
            RoundedRectangle(cornerRadius: 8)
                .fill(.black.opacity(0.8))
        )
        .onAppear {
            animator.startIdleLoop()
        }
    }
}

struct ScanlineOverlay: View {
    @State private var offset: CGFloat = 0

    var body: some View {
        GeometryReader { geo in
            VStack(spacing: 2) {
                ForEach(0..<30, id: \.self) { _ in
                    Rectangle()
                        .fill(.white.opacity(0.05))
                        .frame(height: 1)
                }
            }
            .offset(y: offset)
            .onAppear {
                withAnimation(.linear(duration: 3).repeatForever(autoreverses: false)) {
                    offset = 4
                }
            }
        }
    }
}

struct FaceOutline: Shape {
    func path(in rect: CGRect) -> Path {
        Path(ellipseIn: rect.insetBy(dx: 6, dy: 3))
    }
}

struct BrowShape: View {
    let raise: CGFloat  // 0 to 1
    let furrow: CGFloat // 0 to 1

    var body: some View {
        Path { path in
            path.move(to: CGPoint(x: 0, y: 4 - raise * 3 + furrow * 2))
            path.addQuadCurve(
                to: CGPoint(x: 10, y: 2 - raise * 2)),
                control: CGPoint(x: 5, y: 0 - raise * 4 + furrow * 3)
            )
        }
        .stroke(.white.opacity(0.6), lineWidth: 1.5)
        .frame(width: 10, height: 8)
    }
}

// MARK: - Face Animator

@MainActor
class FaceAnimator: ObservableObject {
    @Published var currentViseme: Viseme = .sil
    @Published var eyeOpenness: CGFloat = 1.0
    @Published var gazeDirection: CGPoint = .zero
    @Published var browRaise: CGFloat = 0
    @Published var browFurrow: CGFloat = 0
    @Published var mouthSmile: CGFloat = 0.1

    private var blinkTimer: Timer?
    private var idleTimer: Timer?

    func startIdleLoop() {
        // Random blinks
        scheduleNextBlink()

        // Subtle gaze drift
        idleTimer = Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { [weak self] _ in
            Task { @MainActor in
                guard let self = self else { return }
                withAnimation(.easeInOut(duration: 1.5)) {
                    self.gazeDirection = CGPoint(
                        x: CGFloat.random(in: -0.3...0.3),
                        y: CGFloat.random(in: -0.2...0.2)
                    )
                }
            }
        }
    }

    private func scheduleNextBlink() {
        let delay = Double.random(in: 2.0...5.0)
        blinkTimer = Timer.scheduledTimer(withTimeInterval: delay, repeats: false) { [weak self] _ in
            Task { @MainActor in
                self?.performBlink()
            }
        }
    }

    private func performBlink() {
        withAnimation(.easeIn(duration: 0.05)) {
            eyeOpenness = 0.0
        }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.08) { [weak self] in
            withAnimation(.easeOut(duration: 0.1)) {
                self?.eyeOpenness = 1.0
            }
            self?.scheduleNextBlink()
        }
    }

    func setExpression(_ expression: Expression) {
        withAnimation(.easeInOut(duration: 0.3)) {
            switch expression {
            case .calm:
                browRaise = 0; browFurrow = 0; mouthSmile = 0.1
            case .thinking:
                browRaise = 0.3; browFurrow = 0.2; mouthSmile = 0
            case .alert:
                browRaise = 0.5; browFurrow = 0; mouthSmile = -0.1
            case .alarmed:
                browRaise = 0.7; browFurrow = 0.5; mouthSmile = -0.3
            case .happy:
                browRaise = 0.2; browFurrow = 0; mouthSmile = 0.5
            case .speaking:
                browRaise = 0.1; browFurrow = 0; mouthSmile = 0
            case .listening:
                browRaise = 0.2; browFurrow = 0.1; mouthSmile = 0.05
            }
        }
    }
}
```

> **Key insight:** The pure SwiftUI approach has a unique advantage for the VIKI aesthetic — you can draw a stylized, holographic face with scan lines, glow effects, and translucent geometry that would be harder to achieve with a pre-made 3D model. The downside is that complex path animations at 60fps can cause SwiftUI body recomputation overhead. Keep the animated paths simple.

---

## Pattern 5: TTS-to-Viseme Pipeline

The lip sync pipeline is independent of the rendering approach. Whether you're using SceneKit, SpriteKit, or SwiftUI, you need to convert speech audio into a stream of viseme events.

### Approach A: Apple AVSpeechSynthesizer with Word-Level Timing

Apple's TTS gives you word-level callbacks but not phoneme-level. You need to estimate visemes from the text.

```swift
import AVFoundation

class TTSVisemeDriver: NSObject, AVSpeechSynthesizerDelegate, ObservableObject {
    private let synthesizer = AVSpeechSynthesizer()
    private var currentUtterance: AVSpeechUtterance?
    private var fullText: String = ""

    // Callback for each viseme
    var onViseme: ((Viseme, TimeInterval) -> Void)?
    var onExpressionHint: ((Expression) -> Void)?

    override init() {
        super.init()
        synthesizer.delegate = self
    }

    func speak(_ text: String) {
        fullText = text
        let utterance = AVSpeechUtterance(string: text)
        utterance.voice = AVSpeechSynthesisVoice(language: "en-US")
        utterance.rate = AVSpeechUtteranceDefaultSpeechRate
        utterance.pitchMultiplier = 1.0
        currentUtterance = utterance
        synthesizer.speak(utterance)
    }

    // MARK: - Delegate

    func speechSynthesizer(
        _ synthesizer: AVSpeechSynthesizer,
        willSpeakRangeOfSpeechString characterRange: NSRange,
        utterance: AVSpeechUtterance
    ) {
        let nsString = fullText as NSString
        let word = nsString.substring(with: characterRange)

        // Convert word to phoneme sequence (simplified)
        let visemes = Self.wordToVisemes(word)

        // Schedule visemes across the estimated word duration
        let estimatedWordDuration = 0.15 * Double(word.count) // rough estimate
        let visemeDuration = estimatedWordDuration / Double(max(visemes.count, 1))

        for (index, viseme) in visemes.enumerated() {
            let delay = Double(index) * visemeDuration
            DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
                self?.onViseme?(viseme, visemeDuration)
            }
        }
    }

    func speechSynthesizer(
        _ synthesizer: AVSpeechSynthesizer,
        didFinish utterance: AVSpeechUtterance
    ) {
        onViseme?(.sil, 0.1)
    }

    // MARK: - Phoneme-to-Viseme Mapping

    /// Simplified English text-to-viseme mapping
    /// For production use, integrate Rhubarb Lip Sync or a phoneme dictionary
    static func wordToVisemes(_ word: String) -> [Viseme] {
        var visemes: [Viseme] = []
        let lower = word.lowercased()
        var i = lower.startIndex

        while i < lower.endIndex {
            let char = lower[i]
            let next = lower.index(after: i) < lower.endIndex
                ? lower[lower.index(after: i)]
                : nil

            switch char {
            case "a": visemes.append(.aa)
            case "e": visemes.append(.ee)
            case "i": visemes.append(.ih)
            case "o": visemes.append(.oh)
            case "u": visemes.append(.ou)
            case "p", "b", "m": visemes.append(.pp)
            case "f", "v": visemes.append(.ff)
            case "t", "d":
                if next == "h" {
                    visemes.append(.th)
                    i = lower.index(after: i)
                } else {
                    visemes.append(.dd)
                }
            case "k", "g": visemes.append(.kk)
            case "s", "z": visemes.append(.ss)
            case "n", "l": visemes.append(.nn)
            case "r": visemes.append(.rr)
            case "c":
                if next == "h" {
                    visemes.append(.ch)
                    i = lower.index(after: i)
                } else {
                    visemes.append(.kk)
                }
            case "j": visemes.append(.ch)
            case "w": visemes.append(.ou)
            case "y": visemes.append(.ih)
            default: break // Skip non-letter characters
            }

            i = lower.index(after: i)
        }

        return visemes.isEmpty ? [.sil] : visemes
    }
}
```

### Approach B: Rhubarb Lip Sync (Offline Pre-Processing)

[Rhubarb Lip Sync](https://github.com/DanielSWolf/rhubarb-lip-sync) is a command-line tool that analyzes audio files and produces frame-accurate viseme data. It uses speech recognition for better accuracy than frequency analysis alone.

```swift
import Foundation

struct RhubarbVisemeEvent: Decodable {
    let start: Double
    let end: Double
    let value: String // A-H mouth shape codes
}

struct RhubarbOutput: Decodable {
    let mouthCues: [RhubarbVisemeEvent]
}

class RhubarbLipSync {
    private let rhubarbPath: String

    init(rhubarbPath: String = "/usr/local/bin/rhubarb") {
        self.rhubarbPath = rhubarbPath
    }

    /// Process an audio file and return viseme timeline
    func process(audioFile: URL, dialogText: String? = nil) async throws -> [RhubarbVisemeEvent] {
        var args = [
            audioFile.path,
            "--machineReadable",
            "-f", "json",
            "--extendedShapes", "GHX" // Include extended mouth shapes
        ]

        // Providing dialog text dramatically improves accuracy
        if let text = dialogText {
            let textFile = FileManager.default.temporaryDirectory
                .appendingPathComponent("dialog.txt")
            try text.write(to: textFile, atomically: true, encoding: .utf8)
            args.append(contentsOf: ["-d", textFile.path])
        }

        let process = Process()
        process.executableURL = URL(fileURLWithPath: rhubarbPath)
        process.arguments = args

        let pipe = Pipe()
        process.standardOutput = pipe

        try process.run()
        process.waitUntilExit()

        let data = pipe.fileHandleForReading.readDataToEndOfFile()
        let output = try JSONDecoder().decode(RhubarbOutput.self, from: data)
        return output.mouthCues
    }

    /// Map Rhubarb mouth shape codes to OVR visemes
    static func rhubarbToViseme(_ code: String) -> Viseme {
        switch code {
        case "A": return .pp   // Closed mouth (m, b, p)
        case "B": return .ee   // Slightly open (most consonants)
        case "C": return .ee   // Open (e, ae)
        case "D": return .aa   // Wide open (a)
        case "E": return .oh   // Rounded (o)
        case "F": return .ou   // Tight rounded (u, w)
        case "G": return .ff   // Upper teeth on lower lip (f, v)
        case "H": return .nn   // Tongue behind teeth (l, th)
        case "X": return .sil  // Idle / silence
        default:  return .sil
        }
    }
}

/// Drives viseme playback from a Rhubarb timeline
class VisemePlayer {
    private var timeline: [RhubarbVisemeEvent] = []
    private var displayLink: CVDisplayLink?
    private var startTime: Double = 0
    private var currentIndex = 0

    var onViseme: ((Viseme) -> Void)?

    func play(timeline: [RhubarbVisemeEvent]) {
        self.timeline = timeline
        currentIndex = 0
        startTime = CACurrentMediaTime()

        // Use a timer instead of CVDisplayLink for simplicity
        Timer.scheduledTimer(withTimeInterval: 1.0/60.0, repeats: true) { [weak self] timer in
            guard let self = self else { timer.invalidate(); return }

            let elapsed = CACurrentMediaTime() - self.startTime

            // Find current viseme based on elapsed time
            while self.currentIndex < self.timeline.count {
                let event = self.timeline[self.currentIndex]
                if elapsed >= event.start && elapsed < event.end {
                    let viseme = RhubarbLipSync.rhubarbToViseme(event.value)
                    self.onViseme?(viseme)
                    break
                } else if elapsed >= event.end {
                    self.currentIndex += 1
                } else {
                    break
                }
            }

            if self.currentIndex >= self.timeline.count {
                timer.invalidate()
                self.onViseme?(.sil)
            }
        }
    }
}
```

### Approach C: Azure Speech Service Viseme API

For the highest quality lip sync, [Azure Speech Service](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-speech-synthesis-viseme) provides real-time blend shape output — 55 facial positions per frame, streamed alongside TTS audio.

```typescript
// Azure Speech SDK — TypeScript example for the pipeline concept
import * as SpeechSDK from 'microsoft-cognitiveservices-speech-sdk';

interface AzureVisemeFrame {
  frameIndex: number;
  blendShapes: number[][]; // 55 blend shape values per frame
}

class AzureVisemeDriver {
  private synthesizer: SpeechSDK.SpeechSynthesizer;
  private visemeQueue: AzureVisemeFrame[] = [];

  constructor(subscriptionKey: string, region: string) {
    const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(
      subscriptionKey,
      region
    );
    speechConfig.speechSynthesisVoiceName = 'en-US-JennyNeural';

    const audioConfig = SpeechSDK.AudioConfig.fromDefaultSpeakerOutput();
    this.synthesizer = new SpeechSDK.SpeechSynthesizer(speechConfig, audioConfig);

    // Subscribe to viseme events
    this.synthesizer.visemeReceived = (sender, event) => {
      if (event.animation) {
        const animationData = JSON.parse(event.animation);
        // animationData.BlendShapes is an array of frames
        // Each frame is an array of 55 float values
        for (const frame of animationData.BlendShapes) {
          this.visemeQueue.push({
            frameIndex: this.visemeQueue.length,
            blendShapes: frame,
          });
        }
      }
    };
  }

  async speak(text: string): Promise<void> {
    this.visemeQueue = [];

    // Use SSML to request blend shapes
    const ssml = `
      <speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis"
             xmlns:mstts="http://www.w3.org/2001/mstts" xml:lang="en-US">
        <voice name="en-US-JennyNeural">
          <mstts:viseme type="FacialExpression"/>
          ${text}
        </voice>
      </speak>
    `;

    return new Promise((resolve, reject) => {
      this.synthesizer.speakSsmlAsync(
        ssml,
        result => {
          if (result.reason === SpeechSDK.ResultReason.SynthesizingAudioCompleted) {
            resolve();
          } else {
            reject(new Error(`Speech synthesis failed: ${result.errorDetails}`));
          }
        },
        error => reject(error)
      );
    });
  }

  // Azure's 55 blend shapes map to ARKit names
  // See: https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-speech-synthesis-viseme
  static readonly AZURE_TO_ARKIT: Record<number, string> = {
    0: 'eyeBlinkLeft',
    1: 'eyeLookDownLeft',
    2: 'eyeLookInLeft',
    3: 'eyeLookOutLeft',
    4: 'eyeLookUpLeft',
    5: 'eyeSquintLeft',
    6: 'eyeWideLeft',
    7: 'eyeBlinkRight',
    8: 'eyeLookDownRight',
    9: 'eyeLookInRight',
    10: 'eyeLookOutRight',
    11: 'eyeLookUpRight',
    12: 'eyeSquintRight',
    13: 'eyeWideRight',
    14: 'jawForward',
    15: 'jawLeft',
    16: 'jawRight',
    17: 'jawOpen',
    18: 'mouthClose',
    19: 'mouthFunnel',
    20: 'mouthPucker',
    21: 'mouthLeft',
    22: 'mouthRight',
    23: 'mouthSmileLeft',
    24: 'mouthSmileRight',
    25: 'mouthFrownLeft',
    26: 'mouthFrownRight',
    27: 'mouthDimpleLeft',
    28: 'mouthDimpleRight',
    29: 'mouthStretchLeft',
    30: 'mouthStretchRight',
    31: 'mouthRollLower',
    32: 'mouthRollUpper',
    33: 'mouthShrugLower',
    34: 'mouthShrugUpper',
    35: 'mouthPressLeft',
    36: 'mouthPressRight',
    37: 'mouthLowerDownLeft',
    38: 'mouthLowerDownRight',
    39: 'mouthUpperUpLeft',
    40: 'mouthUpperUpRight',
    41: 'browDownLeft',
    42: 'browDownRight',
    43: 'browInnerUp',
    44: 'browOuterUpLeft',
    45: 'browOuterUpRight',
    46: 'cheekPuff',
    47: 'cheekSquintLeft',
    48: 'cheekSquintRight',
    49: 'noseSneerLeft',
    50: 'noseSneerRight',
    51: 'tongueOut',
    52: 'headRoll',
    53: 'leftEyeRoll',
    54: 'rightEyeRoll',
  };
}
```

> **Key insight:** Azure's viseme API outputs 55 blend shape values per frame, mapped directly to ARKit blend shape names. This is the highest-fidelity lip sync option available — it handles coarticulation (how sounds blend into each other), emotional modulation, and timing that no rule-based system can match. The cost is API dependency and ~200ms latency.

### Approach D: Real-Time Audio Analysis (wawa-lipsync Style)

For local, zero-latency lip sync, analyze the audio stream's frequency spectrum in real-time.

```swift
import AVFoundation
import Accelerate

class AudioVisemeAnalyzer {
    private let engine = AVAudioEngine()
    private let fftSize = 1024
    private var fftSetup: vDSP_DFT_Setup?

    var onViseme: ((Viseme, CGFloat) -> Void)? // viseme + intensity

    init() {
        fftSetup = vDSP_DFT_zop_CreateSetup(
            nil,
            vDSP_Length(fftSize),
            .FORWARD
        )
    }

    func startAnalyzing(audioNode: AVAudioNode) {
        let format = audioNode.outputFormat(forBus: 0)

        audioNode.installTap(onBus: 0, bufferSize: AVAudioFrameCount(fftSize), format: format) {
            [weak self] buffer, time in
            self?.analyzeBuffer(buffer)
        }

        try? engine.start()
    }

    private func analyzeBuffer(_ buffer: AVAudioPCMBuffer) {
        guard let channelData = buffer.floatChannelData?[0] else { return }
        let frameCount = Int(buffer.frameLength)

        // Calculate RMS energy
        var rms: Float = 0
        vDSP_rmsqv(channelData, 1, &rms, vDSP_Length(frameCount))

        // If silence, emit neutral viseme
        guard rms > 0.01 else {
            DispatchQueue.main.async { self.onViseme?(.sil, 0) }
            return
        }

        // Simple frequency band analysis for viseme estimation
        var realPart = [Float](repeating: 0, count: fftSize)
        var imagPart = [Float](repeating: 0, count: fftSize)

        // Copy input data
        for i in 0..<min(frameCount, fftSize) {
            realPart[i] = channelData[i]
        }

        // Perform FFT
        vDSP_DFT_Execute(fftSetup!, &realPart, &imagPart, &realPart, &imagPart)

        // Calculate magnitude spectrum
        var magnitudes = [Float](repeating: 0, count: fftSize / 2)
        vDSP_zvmags(&realPart, 1, &magnitudes, vDSP_Length(fftSize / 2))

        // Frequency band energy (simplified)
        let sampleRate = 44100.0
        let binWidth = sampleRate / Double(fftSize)

        // Vowel formant ranges (Hz)
        let f1Low = bandEnergy(magnitudes, binWidth: binWidth, lowHz: 200, highHz: 500)
        let f1High = bandEnergy(magnitudes, binWidth: binWidth, lowHz: 500, highHz: 1000)
        let f2Low = bandEnergy(magnitudes, binWidth: binWidth, lowHz: 1000, highHz: 2000)
        let f2High = bandEnergy(magnitudes, binWidth: binWidth, lowHz: 2000, highHz: 4000)
        let highFreq = bandEnergy(magnitudes, binWidth: binWidth, lowHz: 4000, highHz: 8000)

        // Map formant energies to visemes (simplified heuristic)
        let viseme: Viseme
        let intensity = CGFloat(min(rms * 10, 1.0))

        if highFreq > f1Low && highFreq > f2Low {
            viseme = .ss // Sibilants
        } else if f1High > f1Low && f2High > f2Low {
            viseme = .ee // Front vowels
        } else if f1Low > f1High && f2Low > f2High {
            viseme = .oh // Back rounded vowels
        } else if f1Low > 0.5 {
            viseme = .aa // Open vowels
        } else if rms > 0.05 {
            viseme = .dd // Generic consonant
        } else {
            viseme = .sil
        }

        DispatchQueue.main.async {
            self.onViseme?(viseme, intensity)
        }
    }

    private func bandEnergy(_ magnitudes: [Float], binWidth: Double, lowHz: Double, highHz: Double) -> Float {
        let lowBin = Int(lowHz / binWidth)
        let highBin = min(Int(highHz / binWidth), magnitudes.count - 1)
        guard lowBin < highBin else { return 0 }

        var energy: Float = 0
        vDSP_sve(Array(magnitudes[lowBin...highBin]), 1, &energy, vDSP_Length(highBin - lowBin + 1))
        return energy / Float(highBin - lowBin + 1)
    }

    func stop() {
        engine.stop()
    }

    deinit {
        if let setup = fftSetup {
            vDSP_DFT_DestroySetup(setup)
        }
    }
}
```

> **Key insight:** Real-time audio analysis gives you zero-latency lip sync but lower accuracy. It can't distinguish "p" from "b" from "m" — they all look the same in the frequency domain. For a 50x60pt display, this inaccuracy is invisible. For a large display, use Rhubarb or Azure.

---

## Small Examples

### Example 1: Minimal SceneKit Face in SwiftUI

```swift
import SwiftUI
import SceneKit

struct MinimalFaceView: View {
    var body: some View {
        SceneView(
            scene: makeFaceScene(),
            options: [.allowsCameraControl]
        )
        .frame(width: 200, height: 200)
    }

    func makeFaceScene() -> SCNScene {
        let scene = SCNScene()

        // Create a simple face from primitives
        let head = SCNSphere(radius: 0.3)
        head.firstMaterial?.diffuse.contents = NSColor.systemTeal.withAlphaComponent(0.7)
        let headNode = SCNNode(geometry: head)
        scene.rootNode.addChildNode(headNode)

        // Eyes
        for xOffset: Float in [-0.1, 0.1] {
            let eye = SCNSphere(radius: 0.04)
            eye.firstMaterial?.diffuse.contents = NSColor.white
            let eyeNode = SCNNode(geometry: eye)
            eyeNode.position = SCNVector3(xOffset, 0.05, 0.27)
            headNode.addChildNode(eyeNode)

            let pupil = SCNSphere(radius: 0.02)
            pupil.firstMaterial?.diffuse.contents = NSColor.black
            let pupilNode = SCNNode(geometry: pupil)
            pupilNode.position = SCNVector3(0, 0, 0.025)
            eyeNode.addChildNode(pupilNode)
        }

        // Ambient light
        let light = SCNLight()
        light.type = .ambient
        light.color = NSColor(white: 0.6, alpha: 1)
        let lightNode = SCNNode()
        lightNode.light = light
        scene.rootNode.addChildNode(lightNode)

        return scene
    }
}
```

### Example 2: Expression Enum with Blend Shape Presets

```swift
enum Expression: String, CaseIterable {
    case calm, thinking, alert, alarmed, happy, speaking, listening

    static let expressionShapeNames: Set<String> = [
        "browInnerUp", "browDownLeft", "browDownRight",
        "browOuterUpLeft", "browOuterUpRight",
        "eyeSquintLeft", "eyeSquintRight",
        "eyeWideLeft", "eyeWideRight",
        "mouthSmileLeft", "mouthSmileRight",
        "mouthFrownLeft", "mouthFrownRight",
    ]

    static let mouthShapeNames: Set<String> = [
        "jawOpen", "mouthFunnel", "mouthPucker",
        "mouthSmileLeft", "mouthSmileRight",
        "mouthFrownLeft", "mouthFrownRight",
    ]

    var blendShapeTargets: [String: CGFloat] {
        switch self {
        case .calm:
            return ["mouthSmileLeft": 0.1, "mouthSmileRight": 0.1]
        case .thinking:
            return [
                "browInnerUp": 0.3, "eyeSquintLeft": 0.1,
                "eyeSquintRight": 0.1, "mouthPucker": 0.1
            ]
        case .alert:
            return [
                "eyeWideLeft": 0.3, "eyeWideRight": 0.3,
                "browOuterUpLeft": 0.3, "browOuterUpRight": 0.3
            ]
        case .alarmed:
            return [
                "eyeWideLeft": 0.6, "eyeWideRight": 0.6,
                "browInnerUp": 0.5, "browOuterUpLeft": 0.4,
                "browOuterUpRight": 0.4, "mouthFrownLeft": 0.3,
                "mouthFrownRight": 0.3
            ]
        case .happy:
            return [
                "mouthSmileLeft": 0.5, "mouthSmileRight": 0.5,
                "eyeSquintLeft": 0.2, "eyeSquintRight": 0.2
            ]
        case .speaking:
            return ["browInnerUp": 0.1]
        case .listening:
            return [
                "browInnerUp": 0.15, "mouthSmileLeft": 0.05,
                "mouthSmileRight": 0.05
            ]
        }
    }
}
```

### Example 3: Viseme Interpolation with Smoothing

```swift
class VisemeInterpolator {
    private var currentWeights: [String: CGFloat] = [:]
    private var targetWeights: [String: CGFloat] = [:]
    private let smoothingFactor: CGFloat = 0.3 // 0 = instant, 1 = never reach target

    func setTargetViseme(_ viseme: Viseme) {
        targetWeights = viseme.blendShapeTargets
    }

    /// Call this every frame to get smoothed blend shape values
    func tick() -> [String: CGFloat] {
        // Lerp current toward target
        var allKeys = Set(currentWeights.keys).union(targetWeights.keys)

        for key in allKeys {
            let current = currentWeights[key] ?? 0
            let target = targetWeights[key] ?? 0
            currentWeights[key] = current + (target - current) * (1 - smoothingFactor)

            // Snap to zero if close enough
            if abs(currentWeights[key]!) < 0.001 {
                currentWeights[key] = 0
            }
        }

        return currentWeights
    }
}
```

### Example 4: VIKI Holographic Shader (Metal)

```metal
// HolographicFace.metal — Metal shader for VIKI-style translucent face effect

#include <metal_stdlib>
using namespace metal;

struct VertexOut {
    float4 position [[position]];
    float3 worldNormal;
    float3 worldPosition;
    float2 texCoord;
};

fragment float4 holographicFragment(
    VertexOut in [[stage_in]],
    constant float &time [[buffer(0)]],
    constant float3 &cameraPosition [[buffer(1)]]
) {
    // Fresnel rim lighting — glows at edges
    float3 viewDir = normalize(cameraPosition - in.worldPosition);
    float fresnel = 1.0 - abs(dot(viewDir, in.worldNormal));
    fresnel = pow(fresnel, 2.0);

    // Base color — translucent cyan
    float3 baseColor = float3(0.0, 0.8, 1.0);

    // Scanline effect
    float scanline = sin(in.worldPosition.y * 200.0 + time * 2.0) * 0.5 + 0.5;
    scanline = step(0.4, scanline); // Hard-edged scanlines

    // Grid pattern
    float gridX = step(0.95, fract(in.worldPosition.x * 50.0));
    float gridY = step(0.95, fract(in.worldPosition.y * 50.0));
    float grid = max(gridX, gridY) * 0.3;

    // Combine
    float alpha = fresnel * 0.6 + grid + scanline * 0.1;
    alpha = clamp(alpha, 0.0, 0.8);

    float3 color = baseColor * (fresnel + 0.3) + float3(grid * 0.5);

    // Pulse effect
    float pulse = sin(time * 1.5) * 0.1 + 0.9;
    color *= pulse;

    return float4(color, alpha);
}
```

### Example 5: Connecting Status Watcher to Expression

```swift
import Combine

class JaneAvatarBridge: ObservableObject {
    private let statusWatcher: StatusWatcher
    private let faceController: FaceSceneController
    private let ttsDriver: TTSVisemeDriver
    private var cancellables = Set<AnyCancellable>()

    init(statusWatcher: StatusWatcher, faceController: FaceSceneController) {
        self.statusWatcher = statusWatcher
        self.faceController = faceController
        self.ttsDriver = TTSVisemeDriver()

        // Map system status to expressions
        statusWatcher.$currentStatus
            .removeDuplicates()
            .sink { [weak self] status in
                self?.updateExpression(for: status)
            }
            .store(in: &cancellables)

        // Wire TTS visemes to face controller
        ttsDriver.onViseme = { [weak faceController] viseme, duration in
            faceController?.setViseme(viseme)
        }
    }

    private func updateExpression(for status: SystemStatus) {
        let expression: Expression
        switch status {
        case .idle:
            expression = .calm
        case .processing:
            expression = .thinking
        case .alert(let severity):
            expression = severity > 0.7 ? .alarmed : .alert
        case .speaking:
            expression = .speaking
        case .listening:
            expression = .listening
        case .success:
            expression = .happy
        }
        faceController.setExpression(expression)
    }

    func speak(_ text: String) {
        faceController.setExpression(.speaking)
        ttsDriver.speak(text)
    }
}
```

### Example 6: Loading a Ready Player Me Avatar as USDZ

```swift
import Foundation
import SceneKit

class ReadyPlayerMeLoader {
    /// Download and convert a Ready Player Me avatar for SceneKit use
    /// RPM avatars are GLB format — SceneKit needs USDZ or DAE
    static func loadAvatar(
        avatarId: String,
        morphTargets: Bool = true,
        completion: @escaping (Result<SCNScene, Error>) -> Void
    ) {
        var urlString = "https://models.readyplayer.me/\(avatarId).glb"
        if morphTargets {
            urlString += "?morphTargets=ARKit&textureAtlas=512"
        }

        guard let url = URL(string: urlString) else {
            completion(.failure(URLError(.badURL)))
            return
        }

        URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }

            guard let data = data else {
                completion(.failure(URLError(.cannotDecodeContentData)))
                return
            }

            // Save to temp file
            let tempFile = FileManager.default.temporaryDirectory
                .appendingPathComponent("\(avatarId).glb")
            try? data.write(to: tempFile)

            // GLB to SCNScene — requires GLTFKit or Apple's ModelIO
            // Using ModelIO for GLTF support (available since macOS 12)
            do {
                let asset = MDLAsset(url: tempFile)
                let scene = SCNScene(mdlAsset: asset)
                DispatchQueue.main.async {
                    completion(.success(scene))
                }
            } catch {
                DispatchQueue.main.async {
                    completion(.failure(error))
                }
            }
        }.resume()
    }
}
```

### Example 7: NVIDIA Audio2Face-3D Integration Concept

```typescript
// Conceptual integration with NVIDIA's open-source Audio2Face-3D
// The SDK outputs ARKit-compatible blend shapes from audio input
// https://github.com/NVIDIA/Audio2Face-3D

interface Audio2FaceConfig {
  modelPath: string;        // Path to the ONNX model
  sampleRate: number;       // Audio sample rate (16000 Hz typical)
  outputFps: number;        // Target output frame rate
  emotionStrength: number;  // 0-1, how much emotion to add
}

interface Audio2FaceOutput {
  timestamp: number;
  blendShapes: Record<string, number>; // ARKit blend shape names
  emotion: {
    amazement: number;
    anger: number;
    cheekiness: number;
    disgust: number;
    fear: number;
    grief: number;
    joy: number;
    outOfBreath: number;
    pain: number;
    sadness: number;
  };
}

// The Audio2Face-3D SDK is C++ with CUDA dependency
// For Apple Silicon, you'd need to:
// 1. Export the ONNX model
// 2. Convert to Core ML using coremltools
// 3. Run inference via Core ML on the Neural Engine
//
// This is non-trivial but possible for the audio-to-blendshape model.
// The emotion model would also need conversion.

class Audio2FaceCoreMLBridge {
  // Pseudocode for the Core ML conversion path
  async convertModel(onnxPath: string): Promise<string> {
    // python3 -c "
    //   import coremltools as ct
    //   import onnx
    //   model = onnx.load('audio2face.onnx')
    //   mlmodel = ct.converters.onnx.convert(model)
    //   mlmodel.save('audio2face.mlpackage')
    // "
    return 'audio2face.mlpackage';
  }
}
```

### Example 8: Complete Integration — Jane Avatar Controller

```swift
import SwiftUI
import Combine

/// The main orchestrator that ties everything together
@MainActor
class JaneController: ObservableObject {
    // Rendering backend — swap between approaches
    enum RenderingBackend {
        case sceneKit    // Full 3D with blend shapes
        case spriteKit   // 2D layered sprites
        case swiftUI     // Pure SwiftUI shapes
    }

    @Published var backend: RenderingBackend = .swiftUI

    // Sub-controllers
    let spriteScene = AvatarSpriteScene(size: CGSize(width: 100, height: 120))
    let faceController = FaceSceneController()
    let swiftUIAnimator = FaceAnimator()

    // Lip sync
    private let ttsDriver = TTSVisemeDriver()
    private let audioAnalyzer = AudioVisemeAnalyzer()

    // State
    @Published var currentExpression: Expression = .calm
    @Published var isSpeaking = false

    init() {
        // Wire up TTS viseme callbacks
        ttsDriver.onViseme = { [weak self] viseme, duration in
            Task { @MainActor in
                self?.applyViseme(viseme)
            }
        }
    }

    func setExpression(_ expression: Expression) {
        currentExpression = expression

        switch backend {
        case .sceneKit:
            faceController.setExpression(expression)
        case .spriteKit:
            spriteScene.setExpression(expression)
        case .swiftUI:
            swiftUIAnimator.setExpression(expression)
        }
    }

    func speak(_ text: String) {
        isSpeaking = true
        setExpression(.speaking)
        ttsDriver.speak(text)
    }

    private func applyViseme(_ viseme: Viseme) {
        switch backend {
        case .sceneKit:
            faceController.setViseme(viseme)
        case .spriteKit:
            spriteScene.setViseme(viseme)
        case .swiftUI:
            swiftUIAnimator.currentViseme = viseme
        }
    }
}
```

---

## The VIKI Aesthetic

VIKI (Virtual Interactive Kinetic Intelligence) from *I, Robot* (2004) was created by [Digital Domain](https://www.awn.com/vfxworld/i-robot-and-future-digital-effects) using a combination of CGI techniques. The face is characterized by:

- **Translucent, holographic appearance** — the face seems made of light, not solid matter
- **Particle/grid substrate** — visible geometric structure beneath the face surface
- **Edge glow / fresnel lighting** — edges of the face glow brighter than the center
- **Scanline artifacts** — horizontal lines suggesting a display or projection
- **Minimal features** — eyes, mouth, and face outline are the primary readable elements
- **Cool color palette** — cyan, blue, white against dark backgrounds

To recreate this in a native macOS app, you don't need a complex 3D pipeline. The aesthetic is actually *easier* to achieve with 2D techniques:

```swift
struct VIKIFaceView: View {
    @StateObject private var animator = FaceAnimator()

    var body: some View {
        TimelineView(.animation) { timeline in
            let time = timeline.date.timeIntervalSinceReferenceDate

            Canvas { context, size in
                let cx = size.width / 2
                let cy = size.height / 2

                // Background glow
                let glowGradient = Gradient(colors: [
                    .cyan.opacity(0.05),
                    .clear
                ])
                context.fill(
                    Path(ellipseIn: CGRect(
                        x: cx - 30, y: cy - 35,
                        width: 60, height: 70
                    )),
                    with: .radialGradient(
                        glowGradient,
                        center: CGPoint(x: cx, y: cy),
                        startRadius: 10,
                        endRadius: 40
                    )
                )

                // Scanlines
                for y in stride(from: 0.0, to: size.height, by: 3) {
                    let opacity = 0.03 + 0.02 * sin(y * 0.5 + time * 2)
                    var path = Path()
                    path.move(to: CGPoint(x: 0, y: y))
                    path.addLine(to: CGPoint(x: size.width, y: y))
                    context.stroke(
                        path,
                        with: .color(.cyan.opacity(opacity)),
                        lineWidth: 0.5
                    )
                }

                // Face outline — fresnel glow
                let faceRect = CGRect(x: cx - 18, y: cy - 22, width: 36, height: 44)
                let faceOutline = Path(ellipseIn: faceRect)
                context.stroke(
                    faceOutline,
                    with: .color(.cyan.opacity(0.4 + 0.1 * sin(time * 1.5))),
                    lineWidth: 1.5
                )

                // Grid pattern over face
                for gridY in stride(from: faceRect.minY, to: faceRect.maxY, by: 4) {
                    var line = Path()
                    line.move(to: CGPoint(x: faceRect.minX + 5, y: gridY))
                    line.addLine(to: CGPoint(x: faceRect.maxX - 5, y: gridY))
                    context.stroke(
                        line,
                        with: .color(.cyan.opacity(0.08)),
                        lineWidth: 0.5
                    )
                }

                // Eyes
                let eyeY = cy - 6
                for eyeX in [cx - 7.0, cx + 7.0] {
                    let openness = animator.eyeOpenness
                    let eyeWidth: CGFloat = 7
                    let eyeHeight: CGFloat = 4 * openness

                    // Eye glow
                    let eyeRect = CGRect(
                        x: eyeX - eyeWidth/2,
                        y: eyeY - eyeHeight/2,
                        width: eyeWidth,
                        height: eyeHeight
                    )
                    context.fill(
                        Path(ellipseIn: eyeRect),
                        with: .color(.cyan.opacity(0.6))
                    )

                    // Pupil
                    if openness > 0.3 {
                        let pupilSize: CGFloat = 2
                        let pupilRect = CGRect(
                            x: eyeX - pupilSize/2 + animator.gazeDirection.x * 1.5,
                            y: eyeY - pupilSize/2 + animator.gazeDirection.y * 1,
                            width: pupilSize,
                            height: pupilSize
                        )
                        context.fill(
                            Path(ellipseIn: pupilRect),
                            with: .color(.white.opacity(0.9))
                        )
                    }
                }

                // Mouth — viseme-dependent
                drawMouth(
                    context: &context,
                    center: CGPoint(x: cx, y: cy + 10),
                    viseme: animator.currentViseme,
                    time: time
                )
            }
            .frame(width: 50, height: 60)
        }
        .onAppear {
            animator.startIdleLoop()
        }
    }

    private func drawMouth(
        context: inout GraphicsContext,
        center: CGPoint,
        viseme: Viseme,
        time: Double
    ) {
        let (width, height) = viseme.mouthDimensions

        var path = Path()
        if height < 1 {
            // Closed mouth — just a line
            path.move(to: CGPoint(x: center.x - width, y: center.y))
            path.addLine(to: CGPoint(x: center.x + width, y: center.y))
            context.stroke(
                path,
                with: .color(.cyan.opacity(0.5)),
                lineWidth: 1
            )
        } else {
            // Open mouth — ellipse
            let rect = CGRect(
                x: center.x - width,
                y: center.y - height/2,
                width: width * 2,
                height: height
            )
            path.addEllipse(in: rect)
            context.fill(
                path,
                with: .color(.cyan.opacity(0.2))
            )
            context.stroke(
                path,
                with: .color(.cyan.opacity(0.5)),
                lineWidth: 0.8
            )
        }
    }
}

extension Viseme {
    /// Mouth dimensions for 2D rendering (width, height in points)
    var mouthDimensions: (CGFloat, CGFloat) {
        switch self {
        case .sil: return (4, 0)
        case .pp:  return (3, 0)
        case .ff:  return (4, 1)
        case .th:  return (4, 1.5)
        case .dd:  return (4, 2)
        case .kk:  return (3.5, 2.5)
        case .ch:  return (5, 2)
        case .ss:  return (4, 1)
        case .nn:  return (4, 1.5)
        case .rr:  return (3, 2)
        case .aa:  return (5, 4)
        case .ee:  return (6, 2)
        case .ih:  return (4.5, 2)
        case .oh:  return (3.5, 3.5)
        case .ou:  return (3, 3)
        }
    }
}
```

The VIKI aesthetic actually works *better* at small sizes than photorealistic 3D. The simplified, high-contrast, emissive style reads clearly even at 50x60pt. You lose nothing by going stylized — you gain readability.

---

## Comparisons

### Face Rendering Engines

| Engine | Native macOS | Programmatic Control | Lip Sync | Small Size Performance | License | Best For |
|--------|:---:|:---:|:---:|:---:|---------|---------|
| **SceneKit** (Apple) | Yes | Full blend shape API | Manual (you map visemes) | Excellent | Free (Apple SDK) | Production macOS apps with 3D faces |
| **RealityKit** (Apple) | Partial (visionOS focus) | Limited blend shape API | No built-in | Good | Free (Apple SDK) | AR/VR — overkill for 2D HUD |
| **Three.js** via WKWebView | Via WebView | Full morph target API | Via TalkingHead/wawa-lipsync | Moderate (WebGL overhead) | MIT | Rapid prototyping, web-first |
| **SpriteKit** (Apple) | Yes | Frame-based | Manual sprite swapping | Excellent | Free (Apple SDK) | 2D animated avatars, lowest overhead |
| **Live2D Cubism SDK** | C++ (no Swift bindings) | Full parameter API | Manual (parameter mapping) | Good | Proprietary ($) | VTuber-style 2D avatars |
| **Unity (as library)** | Via Unity as a Library | Full API | Via plugins | Heavy (~200MB runtime) | Per-seat license | When you're already using Unity |
| **Metal** (custom) | Yes | Whatever you build | Whatever you build | Best possible | Free (Apple SDK) | Maximum control, VIKI-style effects |

### Lip Sync Solutions

| Solution | Real-Time | Accuracy | Platform | License | Integration Effort |
|----------|:---------:|:--------:|----------|---------|:------------------:|
| **Rhubarb Lip Sync** | No (offline) | High | CLI (macOS/Linux/Win) | MIT | Medium — parse JSON output |
| **Azure Speech Viseme API** | Yes (streaming) | Highest | Cloud API | Pay-per-use | Low — SDK available |
| **AVSpeechSynthesizer** | Yes (word-level) | Low (word-level only) | macOS/iOS native | Free | Low — delegate callbacks |
| **wawa-lipsync** | Yes | Medium | Browser (JS/TS) | MIT | Low (web) / Medium (native) |
| **NVIDIA Audio2Face-3D** | Yes | Very High | CUDA (no Apple Silicon) | Open source (MIT) | High — CUDA dependency |
| **Audio FFT analysis** | Yes | Low | Any | N/A (build yourself) | Medium — signal processing |
| **Oculus OVR LipSync** | Yes | High | Quest/PC (no macOS) | Meta SDK license | N/A for macOS |

### Cloud Avatar APIs

| Provider | Real-Time Streaming | Embed Widget | Programmatic Expression | Pricing | Status |
|----------|:-------------------:|:------------:|:-----------------------:|---------|--------|
| **HeyGen** | Yes (WebRTC) | Yes (SDK) | Yes (API) | $0.10/min streaming | Active |
| **Simli** | Yes (WebSocket) | Yes (React SDK) | Yes (audio-to-video) | Pay-per-use | Active |
| **D-ID** | Yes | Yes (web) | Limited | $0.025/sec | Active |
| **Synthesia** | No (batch) | No | Via API | Enterprise pricing | Active |
| **Soul Machines** | Was yes | Was yes | Was full API | N/A | **Receivership (Feb 2026)** |
| **Avaturn** | No | Yes (iframe SDK) | Avatar creation only | Paid tiers | Active |
| **Ready Player Me** | No (model only) | Yes | Avatar creation only | Free tier + paid | Active |

### Open Source Avatar Projects

| Project | What It Does | Language | macOS Support | Active |
|---------|-------------|----------|:------------:|:------:|
| [TalkingHead](https://github.com/met4citizen/TalkingHead) | Full 3D avatar with lip sync in browser | JS/TS | Via WebView | Yes |
| [Open-LLM-VTuber](https://github.com/Open-LLM-VTuber/Open-LLM-VTuber) | AI assistant with Live2D face | Python | Yes (desktop pet mode) | Yes |
| [openFACS](https://github.com/phuselab/openFACS) | FACS-based 3D face animation | Python | Partial | Limited |
| [FACSvatar](https://github.com/NumesSanguis/FACSvatar) | FACS to avatar pipeline | Python/Unity | Unity only | Limited |
| [wawa-lipsync](https://github.com/wass08/wawa-lipsync) | Real-time browser lip sync | TS | Via WebView | Yes |
| [Audio2Face-3D](https://github.com/NVIDIA/Audio2Face-3D) | Audio to blend shapes | C++/CUDA | No (CUDA required) | Yes |
| [GLTFKit](https://github.com/warrenm/GLTFKit) | GLTF loader for SceneKit/Metal | Obj-C/Swift | Yes | Yes |
| [Rhubarb Lip Sync](https://github.com/DanielSWolf/rhubarb-lip-sync) | Offline audio to viseme | C++ | Yes (CLI) | Yes |

---

## Anti-Patterns

| Don't | Do Instead | Why |
|-------|-----------|-----|
| Embed Unity for a 50x60pt face | Use SceneKit or SpriteKit | Unity adds ~200MB of runtime, separate rendering pipeline, and app startup overhead for rendering a thumbnail-sized face |
| Use ARKit face tracking to drive a synthetic face | Use blend shapes directly via `SCNMorpher` | ARKit tracking requires a TrueDepth camera pointed at a human face — you're generating expressions from code, not tracking them |
| Create 315 pre-composed sprite frames (7 expressions x 15 visemes x 3 blink states) | Use layered sprites with independent mouth/eye/brow layers | Layer composition at runtime reduces asset count from 315 to ~140 and makes adding expressions trivial |
| Set visemes synchronously with TTS word callbacks | Buffer visemes and interpolate between them with a smoothing function | Direct viseme switching creates a "flapping jaw" effect — smoothing makes mouth movement look natural |
| Use photorealistic 3D rendering at small sizes | Use stylized, high-contrast rendering (VIKI aesthetic) | At 100x120 pixels, photorealistic detail is invisible. Stylized faces with emissive edges and clean shapes read better at low resolution |
| Run lip sync at 60fps | Run lip sync at 15-24fps, render at 30fps | Human speech produces about 10-15 visemes per second. 60fps viseme updates waste compute with no visual benefit |
| Put the avatar renderer in the SwiftUI body diff cycle | Use `NSViewRepresentable` or `SpriteView` with imperative updates | SwiftUI will recompute the body on every state change. SceneKit and SpriteKit have their own render loops — let them run independently |
| Use WKWebView for the primary notch-sized avatar | Use WKWebView only for larger avatar displays (panels, full-screen) | WKWebView runs in a separate process with IPC overhead. For a 50x60pt view that's always visible, native rendering is dramatically more efficient |
| Apply expression and lip sync to the same blend shapes without layering | Use a priority-based layer system where lip sync overrides only mouth shapes | Without layering, setting "alarmed" (furrowed brow + frown) while speaking will fight with viseme mouth shapes, causing jitter |
| Generate blend shape models from scratch | Start with Ready Player Me GLB export with `?morphTargets=ARKit` | RPM gives you a properly rigged model with ARKit-compatible blend shapes for free. You can customize the appearance later |

---

## Recommended Architecture for Jane

Based on the analysis above, here is the recommended implementation path for a notch-HUD AI assistant avatar:

### Phase 1: Pure SwiftUI (Ship Fast)

Use the `Canvas`-based VIKI aesthetic approach. No external dependencies, no 3D models needed, renders in the existing SwiftUI HUD with zero integration friction.

```
┌─────────────────────────────────────────┐
│ StatusWatcher  ──→  JaneController      │
│                      │                  │
│              ┌───────┴───────┐          │
│              │               │          │
│        Expression      TTSVisemeDriver  │
│        State Machine         │          │
│              │               │          │
│              └───────┬───────┘          │
│                      │                  │
│               FaceAnimator              │
│                      │                  │
│          VIKIFaceView (Canvas)          │
│            50x60pt in notch             │
└─────────────────────────────────────────┘
```

**Effort:** 2-3 days. Ship it.

### Phase 2: SceneKit Upgrade (When Avatar Goes Larger)

When you add a floating panel or full-screen mode, upgrade to SceneKit with a Ready Player Me model. The existing `FaceAnimator` state feeds into `FaceSceneController` blend shapes.

### Phase 3: TalkingHead WebView (Rich Interactions)

For a future chat/conversation mode where Jane appears full-screen, embed TalkingHead via WKWebView. This gives you the full Ready Player Me ecosystem, Mixamo animations, and real-time lip sync without building the 3D pipeline yourself.

### The Control Surface (Shared Across All Phases)

```swift
protocol JaneFace {
    func setExpression(_ expression: Expression, duration: TimeInterval)
    func setViseme(_ viseme: Viseme)
    func setGaze(_ direction: CGPoint)
    func blink()
    func startIdle()
    func stopIdle()
}

// Each rendering backend conforms to JaneFace
// The JaneController dispatches to whichever backend is active
```

This protocol remains stable across all three phases. The rendering backend is a swappable implementation detail.

---

## Talking Head Models: SadTalker, Wav2Lip, and Audio2Face

These are ML models that generate video of a talking face from audio input. They're fundamentally different from the real-time rendering approaches above — they produce *video frames*, not *controllable 3D state*.

### SadTalker

[SadTalker](https://github.com/OpenTalker/SadTalker) generates talking head video from a single image + audio. It produces 3DMM motion coefficients (head pose + expression) and renders them into video frames.

- **Can it run on Apple Silicon?** Yes, via PyTorch with MPS backend. Inference is not real-time (~5-10 seconds per second of video on M1/M2).
- **Can it render in SwiftUI?** Not directly — it produces video files, not real-time frames.
- **Is it useful for Jane?** No for real-time. Potentially yes for pre-generating avatar video clips that play during long responses.

### Wav2Lip

[Wav2Lip](https://github.com/Rudrabha/Wav2Lip) modifies existing video to match new audio — it's a lip-sync correction tool, not a face generator.

- **Real-time?** No. Produces video files.
- **Useful for Jane?** No.

### NVIDIA Audio2Face-3D

[Audio2Face-3D](https://github.com/NVIDIA/Audio2Face-3D) is the most relevant ML model. It takes audio input and outputs **blend shape weights** (not video frames), which can drive any 3D face. It was [open-sourced in September 2025](https://developer.nvidia.com/blog/nvidia-open-sources-audio2face-animation-model/).

- **Can it run on Apple Silicon?** Not natively — the SDK requires CUDA/TensorRT. However, the ONNX models could theoretically be converted to Core ML using `coremltools`. This is untested but architecturally feasible.
- **Output format:** 52 ARKit-compatible blend shape weights per frame + emotion labels.
- **Is it useful for Jane?** Potentially the best lip sync solution *if* the Core ML conversion works. The model is small enough for real-time inference on the Neural Engine.

> **Key insight:** Audio2Face-3D is the bridge between ML lip sync and real-time 3D rendering. Unlike SadTalker/Wav2Lip (which produce video), Audio2Face outputs blend shape weights that feed directly into SceneKit. If someone ports it to Core ML, it becomes the definitive lip sync solution for macOS avatars.

---

## Expression Control Standards

### ARKit Blend Shapes (52 locations)

The de facto standard. [Full documentation](https://developer.apple.com/documentation/arkit/arfaceanchor/blendshapelocation). Every major avatar platform supports these names.

Categories:
- **Eyes** (14): blink, look direction, squint, wide for each eye
- **Brows** (5): inner up, outer up, down for each brow
- **Jaw** (4): open, forward, left, right
- **Mouth** (19): smile, frown, press, stretch, pucker, funnel, dimple, roll, shrug, lower, upper
- **Cheek** (3): puff, squint left, squint right
- **Nose** (2): sneer left, sneer right
- **Tongue** (1): out
- **Other** (4): head roll, eye roll left/right (Azure extension)

### FACS (Facial Action Coding System)

The scientific standard — 46 Action Units describing individual muscle movements. More granular than ARKit but less directly supported by avatar platforms. [FACS reference](https://www.paulekman.com/facial-action-coding-system/).

Mapping FACS AUs to ARKit blend shapes:

| FACS AU | Name | ARKit Blend Shape |
|---------|------|------------------|
| AU1 | Inner Brow Raise | `browInnerUp` |
| AU2 | Outer Brow Raise | `browOuterUpLeft/Right` |
| AU4 | Brow Lowerer | `browDownLeft/Right` |
| AU5 | Upper Lid Raise | `eyeWideLeft/Right` |
| AU6 | Cheek Raise | `cheekSquintLeft/Right` |
| AU7 | Lid Tightener | `eyeSquintLeft/Right` |
| AU9 | Nose Wrinkle | `noseSneerLeft/Right` |
| AU10 | Upper Lip Raise | `mouthUpperUpLeft/Right` |
| AU12 | Lip Corner Pull (smile) | `mouthSmileLeft/Right` |
| AU15 | Lip Corner Depressor (frown) | `mouthFrownLeft/Right` |
| AU17 | Chin Raise | `mouthShrugLower` |
| AU20 | Lip Stretch | `mouthStretchLeft/Right` |
| AU23 | Lip Tightener | `mouthPressLeft/Right` |
| AU25 | Lips Part | `jawOpen` (partial) |
| AU26 | Jaw Drop | `jawOpen` |
| AU28 | Lip Suck | `mouthRollLower/Upper` |
| AU45 | Blink | `eyeBlinkLeft/Right` |

### OVR Visemes (15 mouth shapes)

Defined by [Meta/Oculus](https://developers.meta.com/horizon/documentation/unity/audio-ovrlipsync-viseme-reference/), these are the standard for lip sync in VR/game engines. Ready Player Me supports both [ARKit and OVR viseme](https://docs.readyplayer.me/ready-player-me/api-reference/avatars/morph-targets/oculus-ovr-libsync) morph targets.

### VTube Studio API Parameters

[VTube Studio](https://github.com/DenchiSoft/VTubeStudio) uses a WebSocket API on `ws://localhost:8001`. Parameters map to Live2D model parameters rather than blend shapes. You can mix face-tracking values with API-driven values using a weight parameter (0-1).

```typescript
// VTube Studio WebSocket message to set a parameter
interface VTSParameterMessage {
  apiName: 'VTubeStudioPublicAPI';
  apiVersion: '1.0';
  requestID: string;
  messageType: 'InjectParameterDataRequest';
  data: {
    faceFound: boolean;
    mode: 'set' | 'add';
    parameterValues: Array<{
      id: string;
      weight?: number; // 0-1, blend with face tracking
      value: number;
    }>;
  };
}

// Example: set mouth open to 50% while letting face tracking control 50%
const message: VTSParameterMessage = {
  apiName: 'VTubeStudioPublicAPI',
  apiVersion: '1.0',
  requestID: 'req-001',
  messageType: 'InjectParameterDataRequest',
  data: {
    faceFound: false,
    mode: 'set',
    parameterValues: [
      { id: 'MouthOpen', weight: 0.5, value: 0.8 },
      { id: 'MouthSmile', value: 0.3 },
      { id: 'EyeOpenLeft', value: 1.0 },
      { id: 'EyeOpenRight', value: 1.0 },
    ],
  },
};
```

---

## Live2D Cubism SDK Deep Dive

The [Live2D Cubism SDK for Native](https://www.live2d.com/en/sdk/download/native/) is a C++ library that renders Live2D models. It's the engine behind VTuber avatars and VTube Studio.

### macOS Support

The SDK supports macOS with OpenGL rendering. As of Cubism 5 SDK, Apple Silicon is supported. However, there are no official Swift bindings — you'd need to create an Objective-C++ bridge.

### Programmatic Control

Live2D models are controlled via named parameters (not blend shapes). Each model defines its own parameter set, but common ones include:

```
ParamAngleX      — Head rotation X (-30 to 30 degrees)
ParamAngleY      — Head rotation Y (-30 to 30 degrees)
ParamAngleZ      — Head rotation Z (-30 to 30 degrees)
ParamEyeLOpen    — Left eye open (0 to 1)
ParamEyeROpen    — Right eye open (0 to 1)
ParamEyeBallX    — Eye gaze X (-1 to 1)
ParamEyeBallY    — Eye gaze Y (-1 to 1)
ParamBrowLY      — Left brow position (-1 to 1)
ParamBrowRY      — Right brow position (-1 to 1)
ParamMouthOpenY  — Mouth open (0 to 1)
ParamMouthForm   — Mouth shape (-1 to 1, frown to smile)
```

### Integration Approach for macOS

```cpp
// Conceptual C++ integration — would need Obj-C++ bridge to Swift
#include "CubismFramework.hpp"
#include "Model/CubismUserModel.hpp"

class JaneLive2DModel : public Csm::CubismUserModel {
public:
    void SetExpression(const char* expressionName) {
        // Load and apply a .exp3.json expression file
        auto expression = LoadExpression(expressionName);
        if (expression) {
            GetExpressionManager()->StartMotion(expression, false, 1.0f);
        }
    }

    void SetMouthOpen(float value) {
        // Direct parameter manipulation
        auto model = GetModel();
        auto paramIndex = model->GetParameterIndex(
            Csm::CubismFramework::GetIdManager()->GetId("ParamMouthOpenY")
        );
        model->SetParameterValue(paramIndex, value);
    }

    void SetEyeBlink(float leftOpen, float rightOpen) {
        auto model = GetModel();
        auto leftIndex = model->GetParameterIndex(
            Csm::CubismFramework::GetIdManager()->GetId("ParamEyeLOpen")
        );
        auto rightIndex = model->GetParameterIndex(
            Csm::CubismFramework::GetIdManager()->GetId("ParamEyeROpen")
        );
        model->SetParameterValue(leftIndex, leftOpen);
        model->SetParameterValue(rightIndex, rightOpen);
    }
};
```

### Should You Use Live2D for Jane?

**Probably not.** Live2D's strengths (deformable 2D meshes, anime-style art, VTuber ecosystem) don't align with the VIKI aesthetic. The C++ SDK requires significant bridging work for Swift integration. If you wanted an anime-style avatar, Live2D + VTube Studio API would be the right choice — but for a holographic AI face, native Apple frameworks give you more control with less friction.

---

## Open-LLM-VTuber: The Closest Existing Project

[Open-LLM-VTuber](https://github.com/Open-LLM-VTuber/Open-LLM-VTuber) is the most complete open-source AI assistant with a visual avatar. It supports:

- Any LLM backend (Ollama, OpenAI, Claude, Gemini, etc.)
- Live2D avatar with lip sync
- Voice interaction with interruption
- Desktop pet mode (floating avatar on screen)
- Vision capability (can see your screen)
- Runs on macOS with Apple Silicon

It's written in Python with a web frontend, so it's not directly embeddable in a Swift app. But it demonstrates the full pipeline:

```
Audio Input → Speech Recognition → LLM → TTS → Lip Sync → Live2D Render
```

If you're prototyping the Jane concept and want to validate the user experience before building native, running Open-LLM-VTuber in desktop pet mode is the fastest path to a working demo.

---

## Performance Considerations for Notch-Sized Rendering

### CPU/GPU Budget

The notch HUD runs continuously — it's an always-visible overlay. Avatar rendering must be nearly invisible in Activity Monitor.

| Approach | CPU Impact | GPU Impact | Memory | Verdict |
|----------|-----------|-----------|---------|---------|
| SwiftUI Canvas at 30fps | ~0.5% | ~0.2% | ~5MB | Best |
| SpriteKit at 30fps | ~0.3% | ~0.3% | ~8MB (textures) | Excellent |
| SceneKit at 30fps | ~1% | ~1% | ~15MB (3D model) | Good |
| WKWebView + Three.js | ~3-5% | ~2-3% | ~80-120MB | Acceptable for panels |
| Metal custom shader | ~0.2% | ~0.5% | ~5MB | Best (most work) |

### Rendering Strategy

```swift
// Adaptive frame rate based on activity
class AdaptiveRenderer {
    enum ActivityLevel {
        case idle       // Blinks only — render at 10fps
        case active     // Expression changes — render at 24fps
        case speaking   // Lip sync — render at 30fps
        case transition // Expression transition — render at 60fps
    }

    var activityLevel: ActivityLevel = .idle {
        didSet {
            updateFrameRate()
        }
    }

    private func updateFrameRate() {
        let fps: Int
        switch activityLevel {
        case .idle: fps = 10
        case .active: fps = 24
        case .speaking: fps = 30
        case .transition: fps = 60
        }
        scnView?.preferredFramesPerSecond = fps
    }
}
```

At `.idle`, the avatar only needs to render when a blink happens (~once every 3-5 seconds). Between blinks, you can skip rendering entirely. This drops the CPU/GPU cost to effectively zero during idle periods.

---

## The Web-Native Bridge: Using WKWebView Strategically

For scenarios where you want the richness of the Three.js ecosystem without building everything natively, a hybrid approach works well:

```swift
class HybridAvatarView: NSView {
    private let nativeOverlay = VIKIFaceLayer()  // Native 2D for notch
    private lazy var webAvatar: WKWebView = {    // Web 3D for expanded view
        // Only created when needed
        return createWebView()
    }()

    enum DisplayMode {
        case notch      // 50x60pt — use native rendering
        case expanded   // 300x400pt — use web rendering
        case fullscreen // Full screen — use web rendering
    }

    var displayMode: DisplayMode = .notch {
        didSet { updateRendering() }
    }

    private func updateRendering() {
        switch displayMode {
        case .notch:
            webAvatar.isHidden = true
            nativeOverlay.isHidden = false
        case .expanded, .fullscreen:
            nativeOverlay.isHidden = true
            webAvatar.isHidden = false
            // Ensure web avatar is loaded
            if webAvatar.url == nil {
                loadWebAvatar()
            }
        }
    }
}
```

This gives you the best of both worlds:
- Native rendering for the always-visible notch (minimal resource usage)
- Full Three.js + TalkingHead for expanded/fullscreen modes (maximum capability)
- A shared control protocol (`JaneFace`) that works with both backends

---

## Cloud Avatar APIs: When to Use Them

The cloud APIs (HeyGen, Simli, D-ID) are designed for a different use case — generating avatar video at scale for marketing, training, and customer service. They're not ideal for a desktop AI assistant because:

1. **Latency**: Even streaming APIs add 200-500ms of latency. For a face that reacts to system state, this delay is noticeable.
2. **Cost**: At $0.10/minute (HeyGen) for continuous rendering, a desktop assistant would cost $150/day.
3. **Dependency**: Your avatar doesn't work offline or when the API is down.
4. **Control**: You can't set arbitrary blend shapes — you send text or audio and get video back.

**When they ARE useful:**
- Pre-generating video clips for specific responses ("Here's what I found...")
- Building a web-based demo of the assistant concept
- When photorealistic quality is essential and cost isn't a concern

### HeyGen Streaming Avatar (For Reference)

```typescript
// HeyGen Streaming Avatar SDK — embeddable in a web page
// https://docs.heygen.com/docs/streaming-avatar-sdk

import StreamingAvatar, {
  AvatarQuality,
  StreamingEvents,
  TaskType,
} from '@heygen/streaming-avatar';

const avatar = new StreamingAvatar({
  token: 'your-access-token',
});

// Start a streaming session
const session = await avatar.createStartAvatar({
  quality: AvatarQuality.Medium,
  avatarName: 'default',
  language: 'en',
});

// Make the avatar speak
await avatar.speak({
  text: 'Hello, I am Jane, your AI assistant.',
  taskType: TaskType.TALK,
});

// Listen for events
avatar.on(StreamingEvents.STREAM_READY, (event) => {
  // event.detail contains the MediaStream
  // Attach to a <video> element
  const videoEl = document.getElementById('avatar-video');
  videoEl.srcObject = event.detail;
});

// Change avatar expression (limited control)
await avatar.speak({
  text: '<break time="500ms"/> I noticed something concerning.',
  taskType: TaskType.TALK,
});
```

### Simli Real-Time Avatar

```typescript
// Simli WebSocket API — audio in, video out
// https://docs.simli.com/

const ws = new WebSocket('wss://api.simli.com/ws');

// Initialize session
ws.send(JSON.stringify({
  type: 'init',
  apiKey: 'your-api-key',
  faceId: 'default-female',
  audioFormat: 'pcm_16000',
}));

// Send audio chunks, receive video frames
ws.onmessage = (event) => {
  if (event.data instanceof Blob) {
    // Video frame — render to canvas
    renderFrame(event.data);
  } else {
    const msg = JSON.parse(event.data);
    if (msg.type === 'ready') {
      // Start sending audio
      startAudioStream(ws);
    }
  }
};

function startAudioStream(ws: WebSocket) {
  // Capture or generate audio and send as PCM chunks
  const audioContext = new AudioContext({ sampleRate: 16000 });
  // ... audio processing pipeline
}
```

---

## The Viseme Pipeline: End-to-End

Here's how all the pieces connect for the complete lip sync pipeline:

```
┌──────────────────────────────────────────────────────────────┐
│                    Text Input ("Hello, how are you?")        │
│                              │                               │
│                    ┌─────────┴─────────┐                     │
│                    │                   │                      │
│              ┌─────▼─────┐    ┌───────▼────────┐             │
│              │   TTS      │    │  Text-to-Phoneme│            │
│              │  Engine    │    │  (for viseme    │            │
│              │            │    │   pre-compute)  │            │
│              └─────┬──────┘    └───────┬────────┘            │
│                    │                   │                      │
│              Audio Stream      Phoneme Timeline              │
│                    │                   │                      │
│              ┌─────▼──────┐    ┌──────▼─────────┐           │
│              │ Audio       │    │ Phoneme-to-     │           │
│              │ Analysis    │    │ Viseme Map      │           │
│              │ (FFT)       │    │ (OVR 15-set)    │           │
│              └─────┬──────┘    └──────┬─────────┘           │
│                    │                   │                      │
│              Energy/        Timed Viseme                      │
│              Frequency      Sequence                         │
│                    │                   │                      │
│              ┌─────▼───────────────────▼─────────┐          │
│              │     Viseme Interpolator            │          │
│              │     (smoothing + coarticulation)   │          │
│              └─────────────┬─────────────────────┘          │
│                            │                                 │
│                    Blend Shape Weights                        │
│                    (per frame)                                │
│                            │                                 │
│              ┌─────────────▼─────────────────────┐          │
│              │     Layer Compositor               │          │
│              │     Expression + Viseme + Idle     │          │
│              └─────────────┬─────────────────────┘          │
│                            │                                 │
│                    Final Blend Shapes                         │
│                            │                                 │
│              ┌─────────────▼─────────────────────┐          │
│              │     Renderer                       │          │
│              │     (SceneKit / SpriteKit / Canvas) │          │
│              └───────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

The dual-path approach (audio analysis + text-to-phoneme) provides both real-time responsiveness and accuracy:

- **Audio FFT** gives you immediate mouth movement that tracks speech energy — the mouth opens and closes in sync with sound, even if the exact shape is wrong.
- **Text-to-phoneme** gives you the correct viseme sequence — the mouth makes the right shapes, but with a slight delay since you need to pre-process the text.

Blending both paths gives you natural-looking lip sync: the overall mouth movement is driven by audio energy (immediate), while the specific mouth shapes are refined by the phoneme timeline (delayed by ~100ms).

---

## Building a Face Model for SceneKit

If you want a custom 3D face (not a Ready Player Me avatar), here's the pipeline:

### Blender Workflow

1. **Model the base mesh** — a simple face with ~2000-5000 vertices is sufficient for a small display. More geometry gives smoother blend shapes but costs more to render.

2. **Create blend shape targets** — in Blender, these are called "Shape Keys." You need at minimum:
   - 15 OVR viseme shapes (mouth positions)
   - 2 eye blink shapes (left + right)
   - 4 eye look shapes (up/down/left/right or in/out per eye)
   - 4-6 brow shapes (inner up, outer up, down per side)
   - That's ~27 shapes minimum

3. **Name them with ARKit conventions** — use the names from `ARFaceAnchor.BlendShapeLocation` exactly. When you export as GLTF/GLB, these names carry through.

4. **Export as GLTF or USDZ** — SceneKit supports both. USDZ is Apple's native format and loads fastest.

### Using the ARKit Blendshape Helper

The [ARKit Blendshape Helper](https://github.com/elijah-atkins/ARKitBlendshapeHelper) Blender addon generates all 52 ARKit blend shapes as shape keys. Install it in Blender, select your face mesh, and it creates target shapes that you can then sculpt to match your face design.

### Minimum Viable Face Model

For the VIKI aesthetic, you don't need realistic skin, hair, or teeth. A stylized geometric face works better:

```
Vertex count: ~1500-3000
Blend shapes: 27 minimum (visemes + blink + gaze + brows)
Materials: 1 (emissive/translucent shader)
Textures: 0 (procedural in Metal shader)
File size: ~200KB as USDZ
```

Compare this to a Ready Player Me avatar (~5-15MB) or a MetaHuman (~500MB+). For a 50x60pt display, the minimal model is the right choice.

---

## Comparison: AI Assistant Avatar Approaches

| Approach | Development Time | Quality at 50x60pt | Quality at Full Screen | Resource Usage | Offline | Customization |
|----------|:----------------:|:-------------------:|:---------------------:|:--------------:|:-------:|:-------------:|
| Pure SwiftUI Canvas (VIKI style) | 2-3 days | Excellent | Good (stylized) | Negligible | Yes | Full |
| SpriteKit layered | 3-5 days + art | Excellent | Limited | Very Low | Yes | Need artist |
| SceneKit + custom model | 1-2 weeks | Good | Excellent | Low | Yes | Full |
| SceneKit + RPM avatar | 3-5 days | Good | Good | Low | Yes | RPM ecosystem |
| WKWebView + TalkingHead | 2-3 days | Moderate | Excellent | Moderate | Yes | Full |
| HeyGen Streaming | 1 day | N/A | Photorealistic | API call | No | Limited |
| Open-LLM-VTuber | 1 day (config) | Good (Live2D) | Good (Live2D) | Moderate | Yes | Community models |

---

## Summary: What to Build

For the Jane notch HUD avatar:

1. **Start with the VIKI Canvas approach** (Pattern 4 + VIKI section). It ships fast, looks great at small sizes, and has zero dependencies. The holographic aesthetic is distinctive and reads well at 100x120 pixels.

2. **Add the TTS viseme pipeline** (Pattern 5, Approach A or D). Start with the simple text-to-viseme mapper for AVSpeechSynthesizer. Upgrade to Rhubarb for pre-recorded audio or audio FFT for real-time analysis.

3. **Define the JaneFace protocol** now, even if you only implement one backend. This keeps your options open for SceneKit, TalkingHead, or any future rendering approach.

4. **Consider Open-LLM-VTuber** for rapid prototyping of the overall assistant experience, even if the final avatar will be native Swift.

5. **Ignore Unity, Unreal, cloud avatar APIs, and talking head ML models** for the notch use case. They solve different problems at different scales.

The face is the easy part. The hard part is what it expresses and when. Build the expression state machine first, render it second.

---

## References

### Apple Frameworks & Documentation

- [SCNMorpher — Apple Developer Documentation](https://developer.apple.com/documentation/scenekit/scnmorpher) — SceneKit blend shape controller for morph target animation
- [ARFaceAnchor.BlendShapeLocation — Apple Developer Documentation](https://developer.apple.com/documentation/arkit/arfaceanchor/blendshapelocation) — The 52 ARKit blend shape definitions (de facto industry standard)
- [SCNView.preferredFramesPerSecond — Apple Developer Documentation](https://developer.apple.com/documentation/scenekit/scnview/preferredframespersecond) — Frame rate control for SceneKit views
- [Animating SceneKit Content — Apple Developer Documentation](https://developer.apple.com/documentation/scenekit/animation/animating_scenekit_content) — Official guide to SceneKit animation
- [AVSpeechSynthesizer — Apple Developer Documentation](https://developer.apple.com/documentation/avfaudio/avspeechsynthesizer) — Apple's text-to-speech engine
- [AVSpeechSynthesizer.MarkerCallback — Apple Developer Documentation](https://developer.apple.com/documentation/avfaudio/avspeechsynthesizer/markercallback) — Callbacks for speech markers during synthesis
- [speechSynthesizer(_:willSpeak:utterance:) — Apple Developer Documentation](https://developer.apple.com/documentation/avfaudio/avspeechsynthesizerdelegate/speechsynthesizer(_:willspeak:utterance:)) — Word-level timing callbacks
- [SpriteView — Apple Developer Documentation](https://developer.apple.com/documentation/spritekit/spriteview) — SwiftUI integration for SpriteKit scenes
- [Extend Speech Synthesis with personal and custom voices — WWDC23](https://developer.apple.com/videos/play/wwdc2023/10033/) — Apple's speech synthesis extensions including SSML and Personal Voice

### Blend Shapes & Face Animation Standards

- [ARKit 52 Facial Blendshapes: The Ultimate Guide](https://pooyadeperson.com/the-ultimate-guide-to-creating-arkits-52-facial-blendshapes/) — Anatomy reference for artists creating ARKit-compatible blend shapes
- [ARKit Face Blendshapes Interactive Reference](https://arkit-face-blendshapes.com/) — Interactive visualization of all 52 ARKit blend shape locations
- [Facial Action Coding System — Wikipedia](https://en.wikipedia.org/wiki/Facial_Action_Coding_System) — Overview of FACS, the scientific standard for facial expression encoding
- [Facial Action Coding System — Paul Ekman Group](https://www.paulekman.com/facial-action-coding-system/) — Official FACS reference from its creator
- [FACS — Carnegie Mellon University](https://www.cs.cmu.edu/~face/facs.htm) — Academic reference for the Facial Action Coding System

### Lip Sync Tools & Libraries

- [Rhubarb Lip Sync — GitHub](https://github.com/DanielSWolf/rhubarb-lip-sync) — Command-line tool for generating 2D mouth animation from voice recordings (MIT license)
- [Oculus OVR LipSync Viseme Reference — Meta Developer Docs](https://developers.meta.com/horizon/documentation/unity/audio-ovrlipsync-viseme-reference/) — The 15-viseme standard used across VR and game engines
- [VRChat Visemes Reference](https://wiki.vrchat.com/wiki/Visemes) — Practical viseme documentation for avatar animation
- [Oculus OVR LipSync Morph Targets — Ready Player Me](https://docs.readyplayer.me/ready-player-me/api-reference/avatars/morph-targets/oculus-ovr-libsync) — How RPM avatars expose OVR viseme blend shapes
- [Azure Speech Service — Get Facial Position with Viseme](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-speech-synthesis-viseme) — Microsoft's viseme API with 55 blend shape output
- [wawa-lipsync — GitHub](https://github.com/wass08/wawa-lipsync) — Open-source real-time lip sync library for web (MIT license)
- [Viseme Cheat Sheet & Interactive IPA Chart](https://melindaozel.com/viseme-cheat-sheet/) — Visual reference for phoneme-to-viseme mapping

### 3D Avatar Platforms & Tools

- [Ready Player Me Developer Docs](https://docs.readyplayer.me/) — Avatar creation platform with SDK for Unity, Unreal, and web
- [TalkingHead — GitHub](https://github.com/met4citizen/TalkingHead) — JavaScript class for real-time lip-sync with full-body 3D avatars (RPM compatible)
- [GLTFKit — GitHub](https://github.com/warrenm/GLTFKit) — Objective-C glTF 2.0 loader and Metal-based renderer for SceneKit
- [ARKit Blendshape Helper — GitHub](https://github.com/elijah-atkins/ARKitBlendshapeHelper) — Blender addon that generates ARKit blend shapes for facial motion capture
- [SkinAndMorph — GitHub](https://github.com/spe3d/SkinAndMorph) — SceneKit sample for blend shape (morph) animation
- [openFACS — GitHub](https://github.com/phuselab/openFACS) — Open source FACS-based 3D face animation system
- [FACSvatar — GitHub](https://github.com/NumesSanguis/FACSvatar) — Open source modular framework from face to FACS-based avatar animation
- [Avaturn Developer Docs](https://docs.avaturn.me/docs/integration/web/sdk/introduction/) — Realistic 3D avatar creator with web SDK

### AI Avatar & Lip Sync ML Models

- [NVIDIA Audio2Face-3D — GitHub](https://github.com/NVIDIA/Audio2Face-3D) — Open-source audio-to-blend-shape animation models and tools
- [NVIDIA Audio2Face-3D SDK — GitHub](https://github.com/NVIDIA/Audio2Face-3D-SDK) — High-performance C++ SDK for Audio2Face inference
- [NVIDIA Open Sources Audio2Face — Blog Post](https://developer.nvidia.com/blog/nvidia-open-sources-audio2face-animation-model/) — Announcement and technical overview of the open-source release
- [Audio2Face-3D v3.0 — Hugging Face](https://huggingface.co/nvidia/Audio2Face-3D-v3.0) — Model weights on Hugging Face
- [SadTalker — GitHub](https://github.com/OpenTalker/SadTalker) — Audio-driven single image talking face animation (CVPR 2023)
- [Audio2Face for iClone — Reallusion](https://www.reallusion.com/iclone/nvidia-omniverse/Audio2Face.html) — Reallusion's integration of NVIDIA Audio2Face

### VTuber & Live2D

- [Live2D Cubism SDK for Native — Documentation](https://docs.live2d.com/en/cubism-sdk-manual/cubism-sdk-for-native/) — Official SDK documentation for C++ integration
- [Live2D Cubism SDK Download](https://www.live2d.com/en/sdk/download/native/) — SDK download page
- [CubismNativeSamples — GitHub](https://github.com/Live2D/CubismNativeSamples) — Official Live2D sample implementations
- [VTube Studio API — GitHub](https://github.com/DenchiSoft/VTubeStudio) — WebSocket API for controlling VTuber models
- [VTubeStudioJS — GitHub](https://github.com/Hawkbat/VTubeStudioJS) — JavaScript implementation of the VTube Studio API
- [Open-LLM-VTuber — GitHub](https://github.com/Open-LLM-VTuber/Open-LLM-VTuber) — Open-source AI assistant with Live2D avatar, voice interaction, and LLM integration

### Cloud Avatar APIs

- [HeyGen Streaming Avatar SDK Documentation](https://docs.heygen.com/docs/streaming-avatar-sdk) — Real-time AI avatar streaming via WebRTC
- [HeyGen API Documentation](https://docs.heygen.com/) — Full API reference for video generation and streaming avatars
- [Simli Documentation](https://docs.simli.com/) — Real-time lip-synced AI avatar API
- [Simli Auto API Reference](https://docs.simli.com/api-reference/simli-auto) — Simplified avatar session management
- [Top 5 Best Avatar APIs 2025 — A2E](https://a2e.ai/top-5-best-avatar-apis-2025/) — Comparison of major avatar API providers
- [Soul Machines Developer Docs](https://docs.soulmachines.com/docs/dp-components) — Documentation for Digital Person components (company in receivership as of Feb 2026)

### Shader & Visual Effects

- [Hologram-Material — GitHub](https://github.com/otanodesignco/Hologram-Material) — GLSL hologram shader for Three.js with fresnel rim lighting
- [Shader Journey: Holograms — Medium](https://medium.com/geekculture/shader-journey-2-holograms-68d4c4a7eb12) — Tutorial on creating holographic shader effects
- [GLSL Hologram Shader — jMonkeyEngine](https://hub.jmonkeyengine.org/t/glsl-hologram-shader/40086) — Discussion and implementation of holographic shaders
- [Custom Metal Drawing in SceneKit — Medium](https://rozengain.medium.com/custom-metal-drawing-in-scenekit-921728e590f1) — How to use custom Metal shaders with SceneKit

### Tutorials & Integration Guides

- [Build a Three.js 3D Avatar with Real-Time AI — Gabber.dev](https://gabber.dev/blog/build-a-threejs-3d-avatar-with-realtime-ai-vision-voice-lip-sync-nextjs) — End-to-end guide for Three.js avatar with lip sync in Next.js
- [Integrating Ready Player Me 3D Model with Lipsyncing in React — Medium](https://medium.com/@israr46ansari/integrating-a-ready-player-me-3d-model-with-lipsyncing-in-react-for-beginners-af5b0c4977cd) — Beginner guide to RPM avatar lip sync
- [Lip Sync Tutorial — Wawa Sensei](https://wawasensei.dev/tuto/react-three-fiber-tutorial-lip-sync) — React Three Fiber lip sync with blend shapes
- [Building a Talking Avatar on the Web — Infosys](https://blogs.infosys.com/emerging-technology-solutions/digital-experience/building-a-talking-avatar-on-the-web-your-guide-to-open-source-digital-humans.html) — Comprehensive guide to open-source digital human avatars
- [Creating Face-Based AR Experiences — Apple Sample Code](https://github.com/DaoKeLegend/Creating-Face-Based-AR-Experiences) — Apple's official ARKit face tracking sample
- [Using SpriteKit to create animations in Swift — Swift by Sundell](https://www.swiftbysundell.com/articles/using-spritekit-to-create-animations-in-swift/) — Guide to SpriteKit animation in Swift
- [How to integrate SpriteKit using SpriteView — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftui/how-to-integrate-spritekit-using-spriteview) — SwiftUI integration for SpriteKit

### Film & Visual Reference

- ['I, Robot' and the Future of Digital Effects — Animation World Network](https://www.awn.com/vfxworld/i-robot-and-future-digital-effects) — Behind-the-scenes look at the VFX of I, Robot including VIKI
- [VIKI — I, Robot Wiki](https://irobot.fandom.com/wiki/VIKI) — Character reference for VIKI's visual design
- [Reallusion iClone Python API](https://www.reallusion.com/iclone/open-api/python/) — Programmatic control of iClone characters
- [Character Creator — Reallusion](https://www.reallusion.com/character-creator/) — Professional 3D character design software
