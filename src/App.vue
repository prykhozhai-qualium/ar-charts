<template>
  <div id="app"></div>
</template>

<script>
import * as THREE from "three";
import { ARButton } from "three/examples/jsm/webxr/ARButton.js";
import { TextGeometry } from "three/examples/jsm/geometries/TextGeometry.js";
import helvetiker_regular from "three/examples/fonts/helvetiker_regular.typeface.json";

import { FontLoader } from "three/examples/jsm/loaders/FontLoader.js";

export default {
  name: "App",
  data() {
    return {
      framestamp: 0,
      framestamp_vector: 1,
      framestamp_scale: Math.PI * 600,
      camera: null,
      scene: null,
      renderer: null,
      animation_frame_callbacks: [],
      reticle: undefined,
      hit: {
        hitTestSource: null,
        hitTestSourceRequested: false,
      },
      objects: {
        grid: {
          text: {
            mesh: null,
          },
        },
      },
      options: {
        scale: 0.5,
        points: {
          width: 60,
          length: 60,
          color: new THREE.Color(0xffffff),
        },
        grid: {
          text: null,
          size: {
            x: 15,
            y: 15,
            z: 15,
          },
        },
      },
    };
  },
  components: {},
  methods: {
    createAnimFunction(x, z) {
      let k_1 =
        (1 / 2) * Math.cos((this.framestamp / this.framestamp_scale) * Math.PI);
      let k_2 =
        (1 / 4) * Math.sin((this.framestamp / this.framestamp_scale) * Math.PI);
      let k_3 =
        (1 / 4) *
        Math.sin(16 * (this.framestamp / this.framestamp_scale) * Math.PI);
      let y =
        2 * Math.cos(k_2 * x + this.framestamp / 15000) +
        2 * Math.cos(k_2 * z + this.framestamp / 15000) +
        5 * k_2 * Math.sin(k_1 * z) +
        6;

      y = 1.5 * (k_3 + 0.25) * y;

      return y;
    },
    setUpSphere(parent) {
      const geometry = new THREE.IcosahedronGeometry(0.1, 3);
      const material = new THREE.MeshPhongMaterial({ color: 0xffffff });
      let mesh = new THREE.InstancedMesh(
        geometry,
        material,
        this.options.points.width * this.options.points.length
      );

      let x_ratio = this.options.grid.size.x / this.options.points.width;
      let z_ratio = this.options.grid.size.z / this.options.points.length;

      let scale_x =
        (this.options.grid.size.x - (1 / x_ratio) * x_ratio) /
        this.options.points.width;
      let scale_z =
        (this.options.grid.size.z - (1 / z_ratio) * z_ratio) /
        this.options.points.length;

      mesh.instanceMatrix;

      this.animation_frame_callbacks.push((timestamp) => {
        let i = 0;

        for (let _x = 0; _x < this.options.points.width; _x++) {
          for (let _z = 0; _z < this.options.points.length; _z++) {
            const matrix = new THREE.Matrix4();
            let z = scale_z * _z;
            let x = scale_x * _x;

            let y = this.createAnimFunction(_x, _z);
            matrix.setPosition(x, y, z);

            let scale = (y / this.options.grid.size.y) * 8;

            matrix.scale(new THREE.Vector3(scale, scale, scale));

            mesh.updateMatrix();
            mesh.setMatrixAt(i, matrix);

            mesh.setColorAt(
              i,
              new THREE.Color(
                _z / this.options.points.length,
                _x / this.options.points.width,
                1
              )
            );

            i++;
            mesh.instanceMatrix.needsUpdate = true;
          }
        }
      });

      mesh.position.x = -this.options.grid.size.x / 2;
      mesh.position.z = -this.options.grid.size.z / 2;

      parent.add(mesh);
    },
    setUpPoints(parent) {
      const geometry = new THREE.BufferGeometry();

      const positions = new Float32Array(
        this.options.points.width * this.options.points.length * 3
      );
      const colors = new Float32Array(
        this.options.points.width * this.options.points.length * 3
      );

      let k = 0;

      let x_ratio = this.options.grid.size.x / this.options.points.width;
      let z_ratio = this.options.grid.size.z / this.options.points.length;

      let scale_x =
        (this.options.grid.size.x - (1 / x_ratio) * x_ratio) /
        this.options.points.width;
      let scale_z =
        (this.options.grid.size.z - (1 / z_ratio) * z_ratio) /
        this.options.points.length;

      for (let _x = 0; _x < this.options.points.width; _x++) {
        for (let _z = 0; _z < this.options.points.length; _z++) {
          const x = _x;
          const y =
            scale_x * Math.sqrt(_x) +
            scale_z * Math.sqrt(_z) +
            scale_z * (_z / 2);
          const z = _z;

          positions[3 * k] = scale_x * x;
          positions[3 * k + 1] = y;
          positions[3 * k + 2] = scale_z * z;

          const intensity = (y + 0.1) * 5;
          colors[3 * k] =
            (this.options.points.color.r * _x) / this.options.points.width;
          colors[3 * k + 1] = 0.5;
          colors[3 * k + 2] =
            (this.options.points.color.b * (this.options.points.color.r * _z)) /
            this.options.points.length;

          k++;
        }
      }

      geometry.setAttribute(
        "position",
        new THREE.BufferAttribute(positions, 3)
      );
      geometry.setAttribute("color", new THREE.BufferAttribute(colors, 3));
      geometry.computeBoundingBox();

      const material = new THREE.PointsMaterial({
        size: 0.004,
        vertexColors: true,
      });

      const points = new THREE.Points(geometry, material);

      points.position.x = -this.options.grid.size.x / 2;
      points.position.z = -this.options.grid.size.z / 2;

      parent.add(points);
    },
    getText(text) {
      let font_loader = new FontLoader();

      font_loader.parse(helvetiker_regular);

      let text_geometry = new TextGeometry(text, {
        font: font_loader.parse(helvetiker_regular),
        size: 0.3,
        height: 0,
        curveSegments: 10,
        bevelEnabled: true,
        bevelThickness: 0.01,
        bevelSize: 0,
        bevelOffset: 0,
        bevelSegments: 2,
      });

      let textMaterial = new THREE.MeshPhongMaterial({
        color: 0xffffff,
        specular: 0xffffff,
      });

      let text_object = new THREE.Mesh(text_geometry, textMaterial);

      return text_object;
    },
    setUpGrid(parent) {
      let text_wrapper = new THREE.Mesh();
      const material = new THREE.LineBasicMaterial({
        color: 0xffffff,
      });

      const points = [];

      for (let _x = 0; _x < this.options.grid.size.x; _x++) {
        if (_x % 2 == 0) {
          let text_x = this.getText(_x.toString());
          text_x.position.x = _x;
          text_x.position.z = this.options.grid.size.z;
          text_x.rotateY(Math.PI / 3);
          text_wrapper.add(text_x);
        }
        for (let _y = 0; _y < this.options.grid.size.y; _y++) {
          points.push(new THREE.Vector3(_x, _y, 0));
        }
        for (let _z = 0; _z < this.options.grid.size.z; _z++) {
          points.push(new THREE.Vector3(_x, 0, _z));
        }
        points.push(new THREE.Vector3(_x, 0, 0));
      }

      for (let _y = 0; _y < this.options.grid.size.y; _y++) {
        if (_y % 2 == 0) {
          let text_y = this.getText(_y.toString());
          text_y.position.y = _y;
          text_y.position.x = this.options.grid.size.x;
          text_y.rotateY(Math.PI / 3);
          text_wrapper.add(text_y);
        }
        for (let _z = 0; _z < this.options.grid.size.z; _z++) {
          points.push(new THREE.Vector3(0, _y, _z));
        }
        for (let _x = 0; _x < this.options.grid.size.x; _x++) {
          points.push(new THREE.Vector3(_x, _y, 0));
        }
        points.push(new THREE.Vector3(0, _y, 0));
      }

      for (let _z = 0; _z < this.options.grid.size.z; _z++) {
        if (_z % 2 == 0) {
          let text_z = this.getText(_z.toString());
          text_z.position.z = _z;
          text_z.position.x = this.options.grid.size.x;
          text_z.rotateY(Math.PI / 3);
          text_wrapper.add(text_z);
        }
        for (let _y = 0; _y < this.options.grid.size.y; _y++) {
          points.push(new THREE.Vector3(0, _y, _z));
        }
        for (let _x = 0; _x < this.options.grid.size.x; _x++) {
          points.push(new THREE.Vector3(_x, 0, _z));
        }
        points.push(new THREE.Vector3(0, 0, _z));
      }

      const geometry = new THREE.BufferGeometry().setFromPoints(points);

      const grid = new THREE.Line(geometry, material);
      grid.position.x = -this.options.grid.size.x / 2;
      grid.position.z = -this.options.grid.size.z / 2;

      text_wrapper.position.x = -this.options.grid.size.x / 2;
      text_wrapper.position.z = -this.options.grid.size.z / 2;

      this.objects.grid.text.mesh = text_wrapper;

      parent.add(text_wrapper);
      parent.add(grid);
    },
    setUpScene() {
      this.camera = new THREE.PerspectiveCamera(
        45,
        window.innerWidth / window.innerHeight,
        0.01,
        60
      );
      this.camera.position.z = 25;
      this.camera.position.x = 25;
      this.camera.position.y = 35;

      this.camera.lookAt(new THREE.Vector3(0, 0, 0));
      this.scene = new THREE.Scene();

      const light = new THREE.HemisphereLight(0xffffff, 0xbbbbff, 1);
      light.position.set(0.5, 1, 0.25);
      this.scene.add(light);

      this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      this.renderer.xr.enabled = true;

      document.body.appendChild(this.renderer.domElement);

      document.body.appendChild(
        ARButton.createButton(this.renderer, { requiredFeatures: ["hit-test"] })
      );

      let controller = this.renderer.xr.getController(0);

      controller.addEventListener("select", () => {
        if (this.reticle && this.reticle.visible) {
          const mesh = new THREE.Mesh();
          this.setUpGrid(mesh);
          this.setUpSphere(mesh);
          this.animation_frame_callbacks.push(() => {
            this.objects.grid.text.mesh.children.forEach((text) => {
              text.lookAt(this.camera.position);
            });
          });
          this.reticle.matrix.decompose(
            mesh.position,
            mesh.quaternion,
            mesh.scale
          );
          mesh.scale.set(0.03, 0.03, 0.03);
          this.scene.add(mesh);
        }
      });

      this.scene.add(controller);

      this.reticle = new THREE.Mesh(
        new THREE.RingGeometry(0.15, 0.2, 32).rotateX(-Math.PI / 2),
        new THREE.MeshBasicMaterial()
      );
      this.reticle.matrixAutoUpdate = false;
      this.reticle.visible = false;

      this.scene.add(this.reticle);

      this.renderer.setAnimationLoop(this.render);
    },
    render(timestamp, frame) {
      let T = this;
      if (this.framestamp == 0) {
        this.framestamp_vector = 1;
      } else if (this.framestamp == this.framestamp_scale) {
        this.framestamp_vector = -1;
      }
      this.framestamp = this.framestamp + this.framestamp_vector;

      T.animation_frame_callbacks.forEach((cb) => {
        cb(timestamp);
      });

      if (frame) {
        const referenceSpace = T.renderer.xr.getReferenceSpace();
        const session = T.renderer.xr.getSession();

        if (T.hit.hitTestSourceRequested === false) {
          session
            .requestReferenceSpace("viewer")
            .then(function (referenceSpace) {
              session
                .requestHitTestSource({ space: referenceSpace })
                .then(function (source) {
                  T.hit.hitTestSource = source;
                });
            });

          session.addEventListener("end", function () {
            T.hit.hitTestSourceRequested = false;
            T.hit.hitTestSource = null;
          });

          T.hit.hitTestSourceRequested = true;
        }

        if (T.hit.hitTestSource) {
          const hitTestResults = frame.getHitTestResults(T.hit.hitTestSource);

          if (hitTestResults.length) {
            const hit = hitTestResults[0];

            T.reticle.visible = true;
            T.reticle.matrix.fromArray(
              hit.getPose(referenceSpace).transform.matrix
            );
          } else {
            T.reticle.visible = false;
          }
        }
      }

      T.renderer.render(T.scene, T.camera);
    },
  },
  mounted() {
    this.setUpScene();

    // const mesh = new THREE.Mesh();

    // this.setUpGrid(mesh);
    // // this.setUpPoints(mesh);
    // this.setUpSphere(mesh);
    // this.animation_frame_callbacks.push(() => {
    //   this.objects.grid.text.mesh.children.forEach((text) => {
    //     text.lookAt(this.camera.position);
    //   });
    //   mesh.rotation.y += 0.005;
    // });

    // mesh.scale.set(1, 1, 1);

    // this.scene.add(mesh);
  },
};
</script>

<style lang="scss">
* {
  margin: 0;
  padding: 0;
  outline: hidden;
}
body {
  background-color: #000;
}
canvas {
  width: 100vw !important;
  height: 100vh !important;
  z-index: -1;
  position: absolute;
  top: 0;
  left: 0;
}
.label {
  color: #fff;
  font-family: sans-serif;
  padding: 2px;
  background: rgba(0, 0, 0, 0.6);
}
</style>
