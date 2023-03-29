<script lang="ts">
  import { onMount } from "svelte";
  import { Pose as _Pose } from "@mediapipe/pose";
  import { POSE_CONNECTIONS as _POSE_CONNECTIONS } from "@mediapipe/pose";
  import { POSE_LANDMARKS as _POSE_LANDMARKS } from "@mediapipe/pose";
  import { POSE_LANDMARKS_LEFT as _POSE_LANDMARKS_LEFT } from "@mediapipe/pose";
  import { POSE_LANDMARKS_RIGHT as _POSE_LANDMARKS_RIGHT } from "@mediapipe/pose";
  import { Camera as _Camera } from "@mediapipe/camera_utils";

  // hack to fix mediapipe import
  const Pose = _Pose || window.Pose;
  const Camera = _Camera || window.Camera;
  const POSE_CONNECTIONS = _POSE_CONNECTIONS || window.POSE_CONNECTIONS;
  const POSE_LANDMARKS = _POSE_LANDMARKS || window.POSE_LANDMARKS;
  const POSE_LANDMARKS_LEFT =
    _POSE_LANDMARKS_LEFT || window.POSE_LANDMARKS_LEFT;
  const POSE_LANDMARKS_RIGHT =
    _POSE_LANDMARKS_RIGHT || window.POSE_LANDMARKS_RIGHT;

  const emojiTimer = {
    0: "0️⃣",
    1: "1️⃣",
    2: "2️⃣",
    3: "3️⃣",
    4: "4️⃣",
    5: "5️⃣",
  };

  let videoElement: HTMLVideoElement;
  let canvasElement: HTMLCanvasElement;
  let canvasCtx: CanvasRenderingContext2D;
  let landmarksCanvasElement: HTMLCanvasElement;
  let landmarksCtx: CanvasRenderingContext2D;
  let snapCanvasElement: HTMLCanvasElement;
  let snapCanvasCtx: CanvasRenderingContext2D;
  let baseRootEl: HTMLDivElement;
  let voiceRecognition: SpeechRecognition | null = null;
  let enabledVoiceSnap = false;
  let timerInterval: ReturnType<typeof setInterval> | null = null;
  let timerSeconds = 0;

  const limbSeq = [
    {
      // neck -> right shoulder
      start: [
        POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
        POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      ],
      end: POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
      color: [255, 0, 0],
    },
    {
      // right shoulder -> right elbow
      start: POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
      end: POSE_LANDMARKS_RIGHT.RIGHT_ELBOW,
      color: [255, 85, 0],
    },
    {
      // right elbow -> right wrist
      start: POSE_LANDMARKS_RIGHT.RIGHT_ELBOW,
      end: POSE_LANDMARKS_RIGHT.RIGHT_WRIST,
      color: [255, 170, 0],
    },
    {
      // neck -> left shoulder
      start: [
        POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
        POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      ],
      end: POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      color: [255, 255, 0],
    },
    {
      // left shoulder -> left elbow
      start: POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      end: POSE_LANDMARKS_LEFT.LEFT_ELBOW,
      color: [170, 255, 0],
    },
    {
      // left elbow -> left wrist
      start: POSE_LANDMARKS_LEFT.LEFT_ELBOW,
      end: POSE_LANDMARKS_LEFT.LEFT_WRIST,
      color: [85, 255, 0],
    },
    {
      // right neck -> right leg
      start: [
        POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
        POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      ],
      end: POSE_LANDMARKS_RIGHT.RIGHT_HIP,
      color: [0, 255, 0],
    },
    {
      // right leg -> right ankle
      start: POSE_LANDMARKS_RIGHT.RIGHT_HIP,
      end: POSE_LANDMARKS_RIGHT.RIGHT_KNEE,
      color: [0, 255, 85],
    },
    {
      // right ankle -> right foot
      start: POSE_LANDMARKS_RIGHT.RIGHT_KNEE,
      end: POSE_LANDMARKS_RIGHT.RIGHT_HEEL,
      color: [0, 255, 170],
    },
    {
      // left neck -> left leg
      start: [
        POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
        POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      ],
      end: POSE_LANDMARKS_LEFT.LEFT_HIP,
      color: [0, 255, 255],
    },
    {
      // left leg -> left knee
      start: POSE_LANDMARKS_LEFT.LEFT_HIP,
      end: POSE_LANDMARKS_LEFT.LEFT_KNEE,
      color: [0, 170, 255],
    },
    {
      // left knee -> left foot
      start: POSE_LANDMARKS_LEFT.LEFT_KNEE,
      end: POSE_LANDMARKS_LEFT.LEFT_HEEL,
      color: [0, 85, 255],
    },
    {
      // neck to nose
      start: [
        POSE_LANDMARKS_RIGHT.RIGHT_SHOULDER,
        POSE_LANDMARKS_LEFT.LEFT_SHOULDER,
      ],
      end: POSE_LANDMARKS.NOSE,
      color: [0, 0, 255],
    },
    {
      // nose to right eye
      start: POSE_LANDMARKS.NOSE,
      end: POSE_LANDMARKS_RIGHT.RIGHT_EYE,
      color: [85, 0, 255],
    },
    {
      // right eye to right ear
      start: POSE_LANDMARKS_RIGHT.RIGHT_EYE,
      end: POSE_LANDMARKS_RIGHT.RIGHT_EAR,
      color: [170, 0, 255],
    },
    {
      // nose to left eye
      start: POSE_LANDMARKS.NOSE,
      end: POSE_LANDMARKS_LEFT.LEFT_EYE,
      color: [255, 0, 255],
    },
    {
      // left eye to left ear
      start: POSE_LANDMARKS_LEFT.LEFT_EYE,
      end: POSE_LANDMARKS_LEFT.LEFT_EAR,
      color: [255, 0, 170],
    },
  ];

  //   const limbSeq = [
  //    [1, 2], [2, 3], [3, 4], // right arm
  //    [1, 5], [5, 6], [6, 7], // left arm
  //    [1, 8], [8, 9], [9, 10], // right torso → right leg
  //    [1, 11], [11, 12], [12, 13], // left torso → left leg
  //    [1, 0], // neck
  //    [0, 14], [14, 16], // right eye
  //    [0, 15], [15, 17] // left eye
  // ];
  const colors = [
    [255, 0, 0], //
    [255, 85, 0], //
    [255, 170, 0], //
    [255, 255, 0], //
    [170, 255, 0], //
    [85, 255, 0], //
    [0, 255, 0], //
    [0, 255, 85], //
    [0, 255, 170], //
    [0, 255, 255], //
    [0, 170, 255], //
    [0, 85, 255], //
    [0, 0, 255], //
    [85, 0, 255], //
    [170, 0, 255], //
    [255, 0, 255], //
    [255, 0, 170], //
    [255, 0, 85],
  ];
  onMount(() => {
    baseRootEl = document.getElementById("canvas-root") as HTMLDivElement;
    if (!baseRootEl._data) baseRootEl._data = { image: null };

    canvasCtx = canvasElement.getContext("2d");

    // offscreen canvas for landmarks
    landmarksCanvasElement = document.createElement("canvas");
    landmarksCtx = landmarksCanvasElement.getContext("2d");
    landmarksCanvasElement.width = canvasElement.width;
    landmarksCanvasElement.height = canvasElement.height;

    const pose = new Pose({
      locateFile: (file) => {
        return `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`;
      },
    });
    pose.setOptions({
      modelComplexity: 1,
      smoothLandmarks: true,
      enableSegmentation: false,
      smoothSegmentation: false,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5,
    });
    pose.onResults(onResults);

    const camera = new Camera(videoElement, {
      onFrame: async () => {
        await pose.send({ image: videoElement });
      },
      width: 720,
      height: 720,
    });
    camera.start();
  });

  function onResults(results) {
    if (!results.poseLandmarks) {
      return;
    }

    // Only overwrite missing pixels.
    canvasCtx.drawImage(
      results.image,
      0,
      0,
      canvasElement.width,
      canvasElement.height
    );
    landmarksCtx.clearRect(
      0,
      0,
      landmarksCanvasElement.width,
      landmarksCanvasElement.height
    );
    drawBodyPose(landmarksCtx, results.poseLandmarks, POSE_CONNECTIONS);

    // draw offscreen canvas to main canvas
    canvasCtx.drawImage(
      landmarksCanvasElement,
      0,
      0,
      canvasElement.width,
      canvasElement.height
    );
  }
  function snapImage() {
    snapCanvasCtx = snapCanvasElement.getContext("2d");
    snapCanvasCtx.save();
    snapCanvasCtx.fillStyle = "black";
    snapCanvasCtx.fillRect(
      0,
      0,
      snapCanvasElement.width,
      snapCanvasElement.height
    );
    snapCanvasCtx.drawImage(
      landmarksCanvasElement,
      0,
      0,
      snapCanvasElement.width,
      snapCanvasElement.height
    );
    snapCanvasCtx.restore();
    baseRootEl._data.image = snapCanvasElement.toDataURL();
  }
  function drawBodyPose(ctx, landmarks, connections) {
    // from https://huggingface.co/spaces/jonigata/PoseMaker/blob/main/static/poseEditor.js
    let stickWidth = 4;
    let imageSize = Math.min(ctx.canvas.width, ctx.canvas.height);
    const width = ctx.canvas.width;
    const height = ctx.canvas.height;

    stickWidth *= imageSize / 512;

    ctx.globalAlpha = 0.6;

    // edge
    const pose = landmarks;

    for (const connection of limbSeq) {
      let start = {};
      if (Array.isArray(connection.start)) {
        const p1 = landmarks[connection.start[0]];
        const p2 = landmarks[connection.start[1]];
        start.x = (p1.x + p2.x) / 2;
        start.y = (p1.y + p2.y) / 2;
      } else {
        start = landmarks[connection.start];
      }
      const p = start;
      const q = landmarks[connection.end];
      if (p == null || q == null) continue;
      const [X0, Y0] = [p.x * width, p.y * height];
      const [X1, Y1] = [q.x * width, q.y * height];
      let angle = Math.atan2(Y1 - Y0, X1 - X0);
      let magnitude = ((X0 - X1) ** 2 + (Y0 - Y1) ** 2) ** 0.5;
      let polygon = new Path2D();
      polygon.ellipse(
        (X0 + X1) / 2,
        (Y0 + Y1) / 2,
        magnitude / 2,
        stickWidth,
        angle,
        0,
        2 * Math.PI
      );
      ctx.fillStyle = `rgb(${connection.color.join(",")})`;
      ctx.fill(polygon);
    }

    ctx.globalAlpha = 1.0;

    // node
    ctx.font = "12px serif";
    for (const connection of limbSeq) {
      const p = landmarks[connection.end];
      if (p == null) continue;
      const [x, y] = [p.x * width, p.y * height];
      ctx.beginPath();
      ctx.arc(x, y, stickWidth, 0, 2 * Math.PI);
      ctx.fillStyle = `rgb(${connection.color.join(",")})`;
      ctx.fill();
      // ctx.fillStyle = "rgb(255,255,255)";
      // ctx.fillText(connection.end, x - 3, y + 4);
    }
  }
  function triggerTimer(seconds = 5) {
    clearInterval(timerInterval);
    timerInterval = null;
    timerSeconds = seconds;
    timerInterval = setInterval(() => {
      timerSeconds--;
      if (timerSeconds <= 0) {
        clearInterval(timerInterval);
        timerInterval = null;
        timerSeconds = 0;
        snapImage();
      }
    }, 1000);
  }
  function toggleVoiceSnap() {
    if (enabledVoiceSnap && voiceRecognition) {
      voiceRecognition.stop();
      voiceRecognition = null;
      enabledVoiceSnap = false;
      console.log("Voice snap disabled");
      return;
    }
    try {
      const SpeechRecognition =
        window.SpeechRecognition || window.webkitSpeechRecognition;
      voiceRecognition = new SpeechRecognition();

      voiceRecognition.continuous = true;
      voiceRecognition.interimResults = true;
      voiceRecognition.lang = "en-US";

      voiceRecognition.start();
      enabledVoiceSnap = true;
      console.log("Voice snap enabled");

      voiceRecognition.onresult = (event) => {
        for (let i = event.resultIndex; i < event.results.length; i++) {
          const transcript = event.results[i][0].transcript
            .trim()
            .toLowerCase();
          console.log(transcript);
          if (event.results[i].isFinal && transcript.includes("snap")) {
            snapImage();
          }
        }
      };

      voiceRecognition.onerror = (error) => {
        console.error(`Speech recognition error: ${error}`);
      };
    } catch (e) {
      console.error(`Speech recognition not supported: ${e}`);
    }
  }
</script>

<div class="relative container">
  <!-- svelte-ignore a11y-media-has-caption -->
  <video bind:this={videoElement} class="hidden" muted playsinline />
  <canvas
    bind:this={canvasElement}
    width="720px"
    height="720px"
    class="w-full"
  />

  <div class="flex absolute bottom-1 left-1 items-end">
    <div class="flex flex-col gap-2">
      <button on:click={snapImage} class="capture-btn">Snap</button>
      <div class="flex gap-2">
        <button on:click={toggleVoiceSnap} class="capture-btn"
          >{enabledVoiceSnap ? "🎤" : "🎤❌"}</button
        >
        <button
          disabled={timerSeconds > 0}
          on:click={() => triggerTimer()}
          class="capture-btn"
        >
          {emojiTimer[timerSeconds]}🕒
        </button>
      </div>
    </div>
    <canvas
      bind:this={snapCanvasElement}
      width="720px"
      height="720px"
      class="w-32"
    />
  </div>
</div>
""

<style lang="postcss" scoped>
  canvas {
    @apply rounded-lg shadow-sm;
  }
  .capture-btn {
    @apply text-black font-bold z-10 bg-slate-50 rounded-lg px-2 py-1 shadow-sm hover:bg-cyan-100;
  }
  .container :global(*) {
    color: black !important;
  }
</style>
