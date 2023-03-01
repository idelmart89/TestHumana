<template>
    <div style="text-align: center">
        <h2 class="has-text-centered has-text-weight-bold is-size-3">
            CAPTURA FACIAL
        </h2>

        <p class="has-text-centered mt-2 subtitle">
            Siga las intrucciones en pantalla y asegúrese que su cara esté bien iluminada
        </p>

        <div class="has-text-centered mt-4 p-2"
             v-if="action"
             style="background-color: #0967b1; color: white">
            <span>{{ action }}</span>
            <span class="icon" v-if="step1 && !step2">
                <i class="fas fa-smile"></i>
            </span>
            <span class="icon" v-if="step1 && step2">
                <i class="fas fa-laugh-wink"></i>
            </span>
        </div>
        <div class="loader mt-4" v-if="show"></div>
        <div class="text-center mt-4">
            <div class="webcam_container">
                <canvas v-show="isPhotoTaken" ref="photoTakenCanvas"></canvas>
                <video id="video-face"
                       v-show="!isPhotoTaken && !isShotPhoto"
                       ref="camera"
                       :width="450"
                       :height="337"
                       autoplay
                       playsinline
                       @play="onPlay"></video>
                <canvas v-show="!isPhotoTaken && !isShotPhoto"
                        id="photoTaken"
                        ref="canvasface"
                        :width="450"
                        :height="337"></canvas>
            </div>
        </div>

        <div class="buttons is-centered mt-4" v-if="acceptPhoto">
            <button @click="acceptPhotoTaken" class="button is-primary">
                Aceptar captura de imagen
            </button>
            <button @click="main" class="button">
                Volver a capturar imagen
            </button>
        </div>
        <div class="field mb-4"
             v-if="tryAgainOption">
            <div class="control">
                <button @click="main" class="button is-fullwidth">
                    Reintentar Login Facial
                </button>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
    import { Variables } from "../../variables";
    import { Component, Vue, Prop } from "vue-property-decorator";
    import * as H from '@vladmandic/human';

    @Component({})
    export default class FaceDetector extends Vue {
        @Prop() readonly start!: boolean;

        private action = "";
        private step1 = false;
        private step2 = false;
        private isPhotoTaken = false;
        private acceptPhoto = false;
        private image = "";
        private isShotPhoto = false;
        private neutral_step = false;
        private smile_step = false;
        private imageFace = "";
        private attemps = 0;
        private tryAgainOption = false;
        private human: any;
        private humanConfig: any;
        private timestamp: any;
        private fps: any;
        private detectionGestures: any;
        private show = true;


        $refs!: {
            recaptcha: HTMLInputElement;
            canvasface: HTMLCanvasElement;
            camera: HTMLVideoElement;
            photoTakenCanvas: HTMLCanvasElement;
        };
        smileInterval: any;
        neutralInterval: any;
        drawInterval: any;

        get pathBase() {
            return `${Variables.ApiBaseUrl}`;
        }

        async mounted() {

            if (this.start !== false) {

                const humanConfig: Partial<H.Config> = { // user configuration for human, used to fine-tune behavior
                    // backend: 'wasm',
                    modelBasePath: 'https://cdn.jsdelivr.net/npm/@vladmandic/human/models',
                    filter: { enabled: true, equalization: false, flip: false },
                    face: { enabled: true, detector: { rotation: false }, mesh: { enabled: true }, attention: { enabled: false }, iris: { enabled: true }, description: { enabled: false }, emotion: { enabled: false }, antispoof: { enabled: true }, liveness: { enabled: true } },
                    body: { enabled: false },
                    // hand: { enabled: true },
                    hand: { enabled: false },
                    object: { enabled: false },
                    segmentation: { enabled: false },
                    gesture: { enabled: true },
                };
                const human = new H.Human(humanConfig); // create instance of human with overrides from user configuration
                this.human = human;
                this.humanConfig = humanConfig;

                const timestamp = { detect: 0, draw: 0, tensors: 0, start: 0 }; // holds information used to calculate performance and possible memory leaks
                const fps = { detectFPS: 0, drawFPS: 0, frames: 0, averageMs: 0 }; // holds calculated fps information for both detect and screen refresh
                this.timestamp = timestamp;
                this.fps = fps;
                this.detectionGestures = ["none"];

                this.main();


            } else {
                this.stopCameraStream();
            }
        }

        async main() { // main entry point

            console.log('human version:', this.human.version, '| tfjs version:', this.human.tf.version['tfjs-core']);
            console.log('platform:', this.human.env.platform, '| agent:', this.human.env.agent);
            console.log('loading...');

            await this.human.load(); // preload all models

            console.log('backend:', this.human.tf.getBackend(), '| available:', this.human.env.backends);
            console.log('models stats:', this.human.models.stats());
            console.log('models loaded:', this.human.models.loaded());
            console.log('environment', this.human.env);
            console.log('initializing...');
            this.createCameraElement();
            await this.human.warmup(); // warmup function to initialize backend for future faster detection
            await this.webCam(); // start webcam
            await this.detectionLoop(); // start detection loop
            await this.drawLoop(); // start draw loop
            await this.step1Validation();
        }
        async webCam() {

            const devices = await this.human.webcam.enumerate();
            const id = devices[0].deviceId; // use first available video source
            const video = document.getElementById("video-face") as HTMLVideoElement;
            await this.human.webcam.start({ element: video, crop: false, width: video.videoWidth, id }); // use human webcam helper methods and associate webcam stream with a dom element
            const canvas = document.getElementById("photoTaken") as HTMLCanvasElement;

            canvas.width = this.human.webcam.width;
            canvas.height = this.human.webcam.height;

        }
        async detectionLoop() { // main detection loop

            const video = document.getElementById("video-face") as HTMLVideoElement;

            if (!video.paused) {
                if (this.timestamp.start === 0) this.timestamp.start = this.human.now();
                // log('profiling data:', await human.profile(dom.video));
                await this.human.detect(video); // actual detection; were not capturing output in a local variable as it can also be reached via human.result
                const tensors = this.human.tf.memory().numTensors; // check current tensor usage for memory leaks
                if (tensors - this.timestamp.tensors !== 0) console.log('allocated tensors:', tensors - this.timestamp.tensors); // printed on start and each time there is a tensor leak
                this.timestamp.tensors = tensors;
                this.fps.detectFPS = Math.round(1000 * 1000 / (this.human.now() - this.timestamp.detect)) / 1000;
                this.fps.frames++;
                this.fps.averageMs = Math.round(1000 * (this.human.now() - this.timestamp.start) / this.fps.frames) / 1000;
                if (this.fps.frames % 100 === 0 && !video.paused) console.log('performance', { ...this.fps, tensors: this.timestamp.tensors });
            }
            this.timestamp.detect = this.human.now();
            requestAnimationFrame(this.detectionLoop); // start new frame immediately
        }
        async drawLoop() { // main screen refresh loop
            const video = document.getElementById("video-face") as HTMLVideoElement;
            const canvas = document.getElementById("photoTaken") as HTMLCanvasElement;

            if (!video.paused) {
                
                let gestures: string[] = Object.values(this.human.result.gesture).map((gesture: any) => gesture.gesture); // flatten all gestures
                for (var i = 0; i < gestures.length; i++) {
                    this.detectionGestures.push(gestures[i]);
                }

                console.log(this.detectionGestures);
                const interpolated = this.human.next(this.human.result); // smoothen result using last-known results
                const processed = await this.human.image(video); // get current video frame, but enhanced with human.filters
                this.human.draw.canvas(processed.canvas as HTMLCanvasElement, canvas);
                const opt: Partial<H.DrawOptions> = {
                    drawBoxes: true, color: 'blue', drawLabels: false, drawPoints: false, drawAttention: false,
                    drawGestures: false, drawGaze: false, fillPolygons: false, useCurves: false, drawPolygons: false
                };
                await this.human.draw.face(canvas, interpolated.face, opt); // draw labels, boxes, lines, etc.
                //   perf(interpolated.performance); // write performance data

                const now = this.human.now();
                this.fps.drawFPS = Math.round(1000 * 1000 / (now - this.timestamp.draw)) / 1000;
                this.timestamp.draw = now;
                setTimeout(this.drawLoop, 100); // use to slow down refresh from max refresh rate to target of 30 fps
            }
            
        }
        getMultipleRandom(arr: any, num: any) {
            const shuffled = [...arr].sort(() => 0.5 - Math.random());

            return shuffled.slice(0, num);
        }
        createCameraElement() {   
            this.show = false;
            this.action = "POR FAVOR COLOQUE MIRE EN DIRECCIÓN A LA CÁMARA Y ESPERE UN RECUADRO AZUL PARA VALIDAR SU IMAGEN";
            this.step1 = false;
            this.step2 = false;
            this.image = "";
            this.imageFace = "";
            this.isPhotoTaken = false;
            this.isShotPhoto = false;
            this.tryAgainOption = false;
            this.acceptPhoto = false;

            if (navigator.mediaDevices.getUserMedia) {
                navigator.mediaDevices
                    .getUserMedia({ video: true })
                    .then((stream) => {
                        this.$refs.camera.srcObject = stream;
                    })
                    .catch(() => {
                        alert("Puede que la webcam este bloqueada en el navegador");
                    });
            } else {
                alert("Su navegador no permite webcam");
            }
        }

        onPlay() {
            const video = document.getElementById("video-face") as HTMLVideoElement;
            const canvas = document.getElementById("photoTaken") as HTMLCanvasElement;

            const displaySize = {
                width: video.videoWidth,
                height: video.videoHeight,
            };

            // faceapi.matchDimensions(canvas, displaySize);
        }


        async step1Validation() {

            this.action = "CIERRE Y ABRA LOS OJOS EN LOS PRÓXIMOS 3 SEGUNDOS";
            let detections = this.human.result;
            this.detectionGestures = ['none'];
            let gest = 0;
            let detection = 0;

            setTimeout(async () => {
                if (detections.face.length > 0) {
                    detection++;
                    

                    if (this.detectionGestures.includes('blink left eye') && this.detectionGestures.includes('blink right eye')) {
                        gest++;
                        this.smile_step = true;
                        this.action = "MIRE HACIA ARRIBA Y LUEGO HACIA EL CENTRO DE LA IMAGEN";
                        this.step2Validation();
                    }
                    else {
                          this.action = "NO HA CUMPLIDO LA VALIDACIÓN, INTENTE NUEVAMENTE";
                            this.neutral_step = false;
                            this.smile_step = false;
                            this.attemps++;
                            this.tryAgainOption = true;
                            if (3 - this.attemps == 0) {
                                this.action =
                                    "Ha realizado los 3 intentos de validación disponibles, por favor intente en otro momento";
                                this.tryAgainOption = false;
                            }
                            this.isShotPhoto = true;
                            this.stopCameraStream();
                    }
                } else {
                    this.action = "NO SE DETECTA UNA PERSONA";
                    console.log(detections);
                }
            }, 5000);
        }



        async step2Validation() {
            this.detectionGestures = ["none"];
            let detections = this.human.result;
            let gest = 0;
            let detection = 0;

            setTimeout(async () => {
                let gestures: string[] = Object.values(this.human.result.gesture).map((gesture: any) => gesture.gesture); // flatten all gestures
                console.log(gestures);

                if (detections.face.length > 0) {
                    detection++;

                    if (this.detectionGestures.includes('looking up') && this.detectionGestures.includes('looking center') && this.human.result.face[0].real >= 0.6) {
                        gest++;
                        this.neutral_step = true;
                    }
                    else {
                        console.log(this.human.result.face[0].live);
                        console.log(this.human.result.face[0].real);
                        
                            this.action = "NO HA CUMPLIDO LA VALIDACIÓN, INTENTE NUEVAMENTE";
                            
                            this.neutral_step = false;
                            this.smile_step = false;
                            this.attemps++;
                            this.tryAgainOption = true;
                            if (3 - this.attemps == 0) {
                                this.action =
                                    "Ha realizado los 3 intentos de validación disponibles, por favor intente en otro momento";
                                this.tryAgainOption = false;
                            }
                            this.isShotPhoto = true;
                            this.stopCameraStream();
                        
                    }
                    if (this.neutral_step && this.smile_step) {

                        
                        this.takePhoto();

                        this.action = "CAPTURA REALIZADA";
                        this.neutral_step = false;
                        this.smile_step = false;
                    }


                } else {
                    this.action = "NO SE DETECTA UNA PERSONA";
                    console.log(detections);
                }
            }, 4000);
        }


        setGreenOval() {
            this.clearCanvas();
            this.drawOval("#00ff00");
        }

        clearCanvas() {
            let canvas = this.$refs.canvasface;
            var ctx: any = canvas.getContext("2d");
            ctx.clearRect(0, 0, canvas.width, canvas.height);
        }

        drawOval(color: any) {
            let canvas = this.$refs.canvasface;
            var ctx: any = canvas.getContext("2d");
            let width = canvas.width;
            let height = canvas.height;
            ctx.beginPath();
            ctx.ellipse(
                canvas.width / 2,
                canvas.height / 2,
                130,
                130,
                0,
                0,
                2 * Math.PI
            );
            ctx.lineWidth = 3;
            ctx.strokeStyle = color;
            ctx.stroke();
            ctx.rect(0, 0, width, height);
            ctx.fillStyle = "rgba(220, 220, 220, 0.8)";
            ctx.fill("evenodd");
            console.log(width / 2);
            console.log(height / 2);
        }

        takePhoto() {
            this.isPhotoTaken = true;
            const canvas = this.$refs.photoTakenCanvas;
            const ctx: any = canvas.getContext("2d");
            canvas.width = this.$refs.camera.width;
            canvas.height = this.$refs.camera.height;
            ctx.drawImage(this.$refs.camera, 0, 0, canvas.width, canvas.height);

            this.imageFace = canvas.toDataURL("image/jpeg");

            ctx.beginPath();
            ctx.ellipse(
                canvas.width / 2,
                canvas.height / 2,
                130,
                130,
                0,
                0,
                2 * Math.PI
            );
            ctx.lineWidth = 3;
            ctx.strokeStyle = "green";
            ctx.stroke();
            ctx.rect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = "rgba(220, 220, 220, 0.8)";
            ctx.fill("evenodd");

            //save capture
            this.image = canvas.toDataURL("image/jpeg");
            this.acceptPhoto = true;
            this.stopCameraStream();
        }

        stopCameraStream() {
            try {
                const video = document.getElementById("video-face") as HTMLVideoElement;
                video.pause();
                (<MediaStream>video.srcObject)
                    .getTracks()
                    .forEach((stream) => stream.stop());
            } catch { }
        }

        acceptPhotoTaken() {
            this.$emit("photoTaken", this.isPhotoTaken, this.imageFace);
            this.acceptPhoto = false;
        }
    }
</script>

<style lang="scss">
@import "~@/assets/main.scss";

@media only screen and (min-width: 1024px) {
  .camera {
    padding: 0.375rem 3rem !important;
  }
}

#photoTaken {
  position: absolute;
  background-color: transparent;
  width: 450px;
  height: 337px;
}

.webcam_container {
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: transparent;
}

#canvas {
  background-color: #ccc;
  max-width: 100%;
  width: 100%;
}

.subtitle {
  font-size: 15px;
}
    .loader {
        border: 10px solid #e8d9d9;
        border-top: 10px solid #3498db;
        border-radius: 50%;
        width: 65px;
        height: 65px;
        animation: spin 2s linear infinite;
        display: inline-block;
    }

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }

        100% {
            transform: rotate(360deg);
        }
    }

    @media only screen and (max-width: 370px) {
        #photoTaken {
            width: 260px;
            height: unset;
        }
    }

    @media only screen and (min-width: 375px) and (max-width: 400px) {
        #photoTaken {
            width: 275px;
            height: unset;
        }
    }

    @media only screen and (min-width: 405px) and (max-width: 430px) {
        #photoTaken {
            width: 310px;
            height: unset;
        }
    }

    @media only screen and (min-width: 800px) and (max-width: 1024px) {
        #photoTaken {
            width: 335px;
            height: unset;
        }
    }
</style>


