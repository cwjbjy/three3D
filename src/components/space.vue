<template>
  <div id="space">
    <div id="cont"></div>
  </div>
</template>

<script>
import * as THREE from "three";
import * as GEOLIB from "geolib";
import { MapControls } from "three/examples/jsm/controls/OrbitControls";
export default {
  name: "space",
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      center: [12.4794497, 41.8769001],
      MAT_BUILDING: null,
    };
  },
  mounted() {
    this.Awake();
  },
  methods: {
    Awake() {
      let cont = document.getElementById("cont");

      this.scene = new THREE.Scene();

      this.camera = new THREE.PerspectiveCamera(
        45,
        window.innerWidth / window.innerHeight,
        1,
        1000
      );
      this.camera.position.set(12,8,4);

      let light0 = new THREE.AmbientLight(0xfafafa, 0.25);
      let light1 = new THREE.PointLight(0xfafafa, 0.4);
      light1.position.set(200, 90, 40);
      let light2 = new THREE.PointLight(0xfafafa, 0.4);
      light2.position.set(200, 90, -40);
      this.scene.add(light0);
      this.scene.add(light1);
      this.scene.add(light2);

      let gh = new THREE.GridHelper(
        60,
        60,
        new THREE.Color(0x555555),
        new THREE.Color(0x333333)
      );
      this.scene.add(gh);

      let axesHelper = new THREE.AxesHelper(100);
      this.scene.add(axesHelper);

      this.renderer = new THREE.WebGLRenderer({ antialias: true });
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.setClearColor(0xb9d3ff, 1); //背景色
      cont.appendChild(this.renderer.domElement);

      this.controls = new MapControls(this.camera, this.renderer.domElement);
      this.controls.dampingFactor = 0.25; //延迟响应时间
      this.controls.maxDistance = 800;

      this.Update();

      this.MAT_BUILDING = new THREE.MeshPhongMaterial({
        color: new THREE.Color(0xb1e90ff),
      });

      this.GetJSON();
    },
    Update() {
      requestAnimationFrame(this.Update);
      this.renderer.render(this.scene, this.camera);
    },
    GetJSON() {
      fetch("/assets/export.geojson").then((res) => {
        res.json().then((data) => {
          this.LoadBuildings(data);
        });
      });
    },
    LoadBuildings(data) {
      let features = data.features;

      for (let i = 0; i < features.length; i++) {
        let fel = features[i];
        if (!fel["properties"]) return;

        if (fel.properties["building"]) {
          this.addBuilding(
            fel.geometry.coordinates,
            fel.properties,
            fel.properties["building:levels"]
          );
        }
      }
    },
    addBuilding(data, info, height = 1) {
      height = height ? height : 1;

      for (let i = 0; i < data.length; i++) {
        let el = data[i];

        let shape = this.genShape(el, this.center);

        let geometry = this.genGeometry(shape, {
          curveSegments: 1,
          depth: 0.05 * height,
          bevelEnabled: false,
        });

        geometry.rotateX(Math.PI / 2);
        geometry.rotateZ(Math.PI);

        let mesh = new THREE.Mesh(geometry, this.MAT_BUILDING);
        this.scene.add(mesh);
      }
    },

    genShape(points, center) {
      let shape = new THREE.Shape();

      for (let i = 0; i < points.length; i++) {
        let elp = points[i];
        elp = this.GPSRelativePosition(elp, center);

        if (i == 0) {
          shape.moveTo(elp[0], elp[1]);
        } else {
          shape.lineTo(elp[0], elp[1]);
        }
      }

      return shape;
    },
    genGeometry(shape, config) {
      let geometry = new THREE.ExtrudeBufferGeometry(shape, config);
      geometry.computeBoundingBox(); //盒子信息
      return geometry;
    },

    GPSRelativePosition(objPosi, centerPosi) {
      let dis = GEOLIB.getDistance(objPosi, centerPosi);
      let bearing = GEOLIB.getRhumbLineBearing(objPosi, centerPosi);
      let x = centerPosi[0] + dis * Math.cos((bearing * Math.PI) / 180);
      let y = centerPosi[1] + dis * Math.sin((bearing * Math.PI) / 180);
      return [-x / 100, y / 100];
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>
