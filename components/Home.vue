<template>
  <div id="container">
    <div class="hero-intro w-full h-full px-3">
      <div class="z-10">
        <h1>ello</h1>
      </div>
    </div>
  </div>
</template>

<script>
  import './Home.css';
  import * as THREE from 'three';
  import vertex from '../assets/shaders/vertex.glsl'
  import fragment from '../assets/shaders/fragment.glsl'
  import vertex1 from '../assets/shadersTubes/vertex.glsl'
  import fragment1 from '../assets/shadersTubes/fragment.glsl'

  export default {
    mounted(){
      const { createNoise2D  } = require('simplex-noise');
      const noise2D = createNoise2D();

      function computeCurl(x, y, z){
        var eps = 0.0001;

        var curl = new THREE.Vector3();

        //Find rate of change in YZ plane
        var n1 = noise2D(x, y + eps, z);
        var n2 = noise2D(x, y - eps, z);
        //Average to find approximate derivative
        var a = (n1 - n2)/(2 * eps);
        var n1 = noise2D(x, y, z + eps);
        var n2 = noise2D(x, y, z - eps);
        //Average to find approximate derivative
        var b = (n1 - n2)/(2 * eps);
        curl.x = a - b;

        //Find rate of change in XZ plane
        n1 = noise2D(x, y, z + eps);
        n2 = noise2D(x, y, z - eps);
        a = (n1 - n2)/(2 * eps);
        n1 = noise2D(x + eps, y, z);
        n2 = noise2D(x - eps, y, z);
        b = (n1 - n2)/(2 * eps);
        curl.y = a - b;

        //Find rate of change in XY plane
        n1 = noise2D(x + eps, y, z);
        n2 = noise2D(x - eps, y, z);
        a = (n1 - n2)/(2 * eps);
        n1 = noise2D(x, y + eps, z);
        n2 = noise2D(x, y - eps, z);
        b = (n1 - n2)/(2 * eps);
        curl.z = a - b;

        return curl;
      }
      class Sketch {
        constructor(options) {
          this.scene = new THREE.Scene();
          this.scene1 = new THREE.Scene();

          this.container = options.dom;
          this.width = this.container.offsetWidth;
          this.height = this.container.offsetHeight;
          this.renderer = new THREE.WebGLRenderer();
          this.raycaster = new THREE.Raycaster()
          this.mouse = new THREE.Vector2()
          this.eMouse = new THREE.Vector2(0,0)
          this.temp = new THREE.Vector2(0,0)
          this.elasticMouse = new THREE.Vector2(0,0)
          this.elasticMouseVel = new THREE.Vector2(0,0)
          this.renderer.setPixelRatio(window.devicePixelRatio);
          this.renderer.setSize(this.width, this.height);
          this.renderer.setClearColor(0x000000, 1);
          this.renderer.physicallyCorrectLights = true;
          this.renderer.outputEncoding = THREE.sRGBEncoding;
          this.renderer.autoClear = false;

          this.container.appendChild(this.renderer.domElement);

          this.cameraGroup = new THREE.Group()
          this.scene.add(this.cameraGroup)
          this.camera = new THREE.PerspectiveCamera(
            70,
            window.innerWidth / window.innerHeight,
            0.001,
            100
          );

          this.camera.position.set(0, 0, 0.75);
          this.cameraGroup.add(this.camera)
          this.clock = new THREE.Clock();
          this.time = 0;
          this.previousTime = 0;

          this.isPlaying = true;

          this.addObjects();
          this.raycast();
          this.resize();
          this.render();
          this.setupResize();
        }

        raycast(){
          this.raycastPlane = new THREE.Mesh(
            new THREE.PlaneGeometry(10,10),
            this.material
          )

          this.light = new THREE.Mesh(
            new THREE.SphereGeometry(0.02,20,20),
            new THREE.MeshBasicMaterial({color:0xa8e6cf})
          )

          this.scene1.add(this.raycastPlane)
          this.scene.add(this.light)

          this.container.addEventListener('mousemove', (event) => {
            this.mouse.x = ( event.clientX / window.innerWidth ) * 2 - 1
            this.mouse.y = - ( event.clientY / window.innerHeight ) * 2 + 1
            this.raycaster.setFromCamera(this.mouse,this.camera)

            this.eMouse.x = event.clientX
            this.eMouse.y = event.clientY

            const intersects = this.raycaster.intersectObjects([this.raycastPlane])
            if(intersects.length > 0){
              let p = intersects[0].point
              this.eMouse.x = p.x
              this.eMouse.y = p.y
            }
          })
        }


        setupResize() {
          window.addEventListener("resize", this.resize.bind(this));
        }

        resize() {
          this.width = this.container.offsetWidth;
          this.height = this.container.offsetHeight;
          this.renderer.setSize(this.width, this.height);
          this.camera.aspect = this.width / this.height;
          this.camera.updateProjectionMatrix();
        }

        addObjects() {
          let that = this;
          this.material = new THREE.ShaderMaterial({
            extensions: {
              derivatives: "#extension GL_OES_standard_derivatives : enable"
            },
            side: THREE.DoubleSide,
            uniforms: {
              time: { value: 0 },
              uLight:{value: new THREE.Vector3(0,0,0)},
              resolution: { type: "v4", value: new THREE.Vector4() },
            },
            vertexShader: vertex,
            fragmentShader: fragment
          });

          this.materialTubes = new THREE.ShaderMaterial({
            extensions: {
              derivatives: "#extension GL_OES_standard_derivatives : enable"
            },
            side: THREE.DoubleSide,
            uniforms: {
              time: { value: 0 },
              uLight:{value: new THREE.Vector3(0,0,0)},
              resolution: { type: "v4", value: new THREE.Vector4() },
            },
            vertexShader: vertex1,
            fragmentShader: fragment1
          });

          this.geometry = new THREE.PlaneGeometry(1, 1, 1, 1);

          for (let i = 0; i < 2; i++) {
            let path = new THREE.CatmullRomCurve3(this.getCurve(new THREE.Vector3(
              Math.random() - 0.5,
              Math.random() - 0.5,
              Math.random() - 0.5
            )))
            let geometry = new THREE.TubeGeometry(path,600,0.005,8,false)

            let curve = new THREE.Mesh(geometry, this.materialTubes);
            this.scene.add(curve);
          }


        }

        getCurve(start){
          let scale = 1
          let points = []
          points.push(start)
          let currentPoint = start.clone()

          for(let i = 0; i < 600; i++){
            let v = computeCurl(currentPoint.x/scale,currentPoint.y/scale,currentPoint.z/scale)
            currentPoint.addScaledVector(v,0.001)

            points.push(currentPoint.clone())
          }
          return points
        }

        render() {
          this.time = this.clock.getElapsedTime()
          const deltaTime = this.time - this.previousTime
          this.previousTime = this.time
          const parallaxX = this.mouse.x * 0.5
          const parallaxY = this.mouse.y * 0.5
          this.cameraGroup.position.x += (parallaxX - this.cameraGroup.position.x) * 0.05 * deltaTime
          this.cameraGroup.position.y += (parallaxY - this.cameraGroup.position.y) * 0.05 * deltaTime

          this.temp.copy(this.eMouse).sub(this.elasticMouse).multiplyScalar(.15)
          this.elasticMouseVel.add(this.temp)
          this.elasticMouseVel.multiplyScalar(.8)
          this.elasticMouse.add(this.elasticMouseVel)
          this.light.position.x = this.elasticMouse.x
          this.light.position.y = this.elasticMouse.y

          this.material.uniforms.uLight.value = this.light.position
          this.materialTubes.uniforms.uLight.value = this.light.position

          this.material.uniforms.time.value = this.time;
          this.materialTubes.uniforms.time.value = this.time;
          requestAnimationFrame(this.render.bind(this));

          this.renderer.clear();
          this.renderer.render(this.scene1 , this.camera);
          this.renderer.clearDepth();

          this.renderer.render(this.scene, this.camera);
        }
      }
      // new Sketch({
      //   dom: document.getElementById("container")
      // });
    }
  }
</script>

<style lang="scss" scoped>

</style>
