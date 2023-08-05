<template>
  <canvas ref="canvasRef"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader"
import * as THREE from "three"
import gsap from "gsap"
import * as dat from "dat.gui"

//LOADER AND LOADING SECTION

//Instantiate the GLTFLoader
const gltfLoader = new GLTFLoader()

//Instantiate the CubeLoader
const cubeTextureLoader = new THREE.CubeTextureLoader()



//GUI
const gui = new dat.GUI()

//CANVAS
let canvasRef = ref();

var controls

//Utilizziamo i valori del viewport per impostare la grandezza del canvas
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

//Aggiungiamo l'event listner per il resize del canvas
window.addEventListener("resize", ()=>{
  //update sizes for the canvas
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  //update the camera aspect ratio
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  //update renderer
  renderer.setSize(sizes.width, sizes.height)
  //HANDLE PIXEL RATIO
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})


//HANDLE FULL SCREEN (Il codice aggiuntivo è per permettere il funzionamento corretto anche con browser Safari)
window.addEventListener("dblclick",()=>{
  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

let scene = new THREE.Scene()

let renderer

//Lights
const directionalLight = new THREE.DirectionalLight("#ffffff",3)
directionalLight.position.set(0.25, 3, -2.25)
directionalLight.castShadow = true
directionalLight.shadow.camera.far = 15
directionalLight.shadow.mapSize.set(1024, 1024)
gui.add(directionalLight, "intensity").min(0).max(10).step(0.001).name("lightIntensity")
gui.add(directionalLight.position, "x").min(-5).max(5).step(0.001).name("light X")
gui.add(directionalLight.position, "y").min(-5).max(5).step(0.001).name("light Y")
gui.add(directionalLight.position, "z").min(-5).max(5).step(0.001).name("light Z")
scene.add(directionalLight)

// const directionalLightCameraHelper = new THREE.CameraHelper(directionalLight.shadow.camera)
// scene.add(directionalLightCameraHelper)


//Many thing can make our model look wrong (lights, shadows, colors, details) , so we are going to learn some of the way to improve the render and how our model looks

//so let's start goig where we instantiate our renderer, and set the physicallyCorrectLights property to true
//In this way :
//1) we have physically correct lights in out scene   
//2) if we are going to based our value on that, we can achieve the same result beetween different software in easy way
//for example if you export from blender, and you have lights in your exported models, you can get the same lights in trheejs


//Then we are going to use an environment maps: 1) for changing background, 2) for use the environment Maps Light as Light for our scene

//Load the environment map
//(The order in which lload the cubeTexture image are px,nx, py,ny, pz,nz)
const environmentMap = cubeTextureLoader.load([
  "/textures/environmentMaps/0/px.jpg",
  "/textures/environmentMaps/0/nx.jpg",
  "/textures/environmentMaps/0/py.jpg",
  "/textures/environmentMaps/0/ny.jpg",
  "/textures/environmentMaps/0/pz.jpg",
  "/textures/environmentMaps/0/nz.jpg",
])

// 2* change the environmentMap encoding property from linear to sRGBEncoding
environmentMap.colorSpace = THREE.SRGBColorSpace

//and apply the environment map on the background of the render
scene.background = environmentMap


//Load our MODELS
gltfLoader.load("/models/FlightHelmet/glTF/FlightHelmet.gltf", (gltf)=>{
  
  gltf.scene.scale.set(10,10,10)
  gltf.scene.position.set(0,-4,0)
  gltf.scene.rotation.y = -Math.PI * 0.25 
  scene.add(gltf.scene)
  
  gui.add(gltf.scene.rotation, "y").min(-Math.PI).max(Math.PI).step(0.001).name("rotation")

  //*1(come here later when you find another *1) and then call the function after the models il loaded into the scene
  updateAllMaterials()
})

//now we can Apply the environment Maps to the model (precisely on the material)

//there is a problem. locate all this material can be quite difficult
//so we can use a "trick". we can create a function that traverse the whole scene (not only the scene of the model we loaded)
//and apply the environment map on each material of each mesh.

// Function for Updating materials based on the environment Maps
const updateAllMaterials = () =>{
  scene.traverse((child)=>{  //the tranverse function "traverse" every object, every child, every child of child and so on
  
  //Set a control to change the material only for Mesh objects (or what we want. but if you don't do this way, it change also the lights and all the other type of objects materials), and only for object that has a MeshStandardMaterial
  if(child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial){

    //and finally now we can apply the environmentMap to the envMap property of the selected objects
    child.material.envMap = environmentMap
    child.material.envMapIntensity = debugObject.envMapIntensity
    child.material.needsUpdate = true

    child.castShadow = true
    child.receiveShadow = true
  }
  
  })
}

//1* (Go Up where is the other 1* symbol)

//now create a debugObject to store property that can't be store anywhere else (because dat.gui requires that it's an object property to tweak) (Note that can be better to create this in the first section of the code, i'll go here just for stay clear)
const debugObject = {}

//let's start adding the envMapIntensity to the debug object
debugObject.envMapIntensity = 5

//then add a tweak on gui
gui.add(debugObject, "envMapIntensity").min(0).max(10).step(0.001).name("EnvMap Intensity").onChange(updateAllMaterials)


//Actually there is a simple method to apply this material to all the environment objects in the scene 
//just go where you have add the cubeTexture to the background and a line below do this:  scene.environment = environmentMap
//in this way you can get rid of the child.material.envMap = environmentMap in the update all materials function 




//NEXT STEP THE RENDER
//our scene it's looking good, but it's not realistic
//so we are going to work on the webGLRender (go below on the onMounted Hook where we instantiate the render)



// CAMERA
let camera = new THREE.PerspectiveCamera(
  75, 
  sizes.width / sizes.height, 
  0.1,
  100  
);
camera.position.set(0,0,3)
scene.add(camera);

const clock = new THREE.Clock()

//Animate Loop
const animate = () =>{
  const elapsedTime = clock.getElapsedTime()

  controls.update()
  
  renderer.render(scene, camera)
}

onMounted(() => {
  
///////////////////////// RENDERER /////////////////////////////
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true
  });
  
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) // in questo modo il max valore di pixel ratio utilizzaro è 2 (altrimenti devi renderizzare troppi pixel, dispiego enorme di potenza gpu)
  //////////////////////// ENCODING //////////////////////

  //Output Encoding
  //The outputEncoding property controls the output render encoding.
  //The default was THREE.LinearEncoding , but we should use THREE.sRGBEncoding (for the exercise purpouse i change the encoding output, but now the default it's sRGBEncoding, so this is not anymore necessary)
  renderer.outputColorSpace = THREE.SRGBColorSpace
  
  //another possible value for the outputEncoding property is THREE.GammaEncoding, which let us play with the gammaFactor property to control the brightness
  //the gamma encoding is a way of storing colors while optimizing how bright and dark value are stored accordingly to the eyes sensitivity
  //when we use the sRGBEncoding, it's like using the GammaEncoding with a default gammaFactor of 2.2, which is a common value

  //Texture Encoding
  //This sRGB/Gamma/Linear encoding can be applied to everything using colors

  //Infact now the environment map colors are wrong (gray and toned down), and this is because
  //actually we have set the outputEncoding to sRGBEncoding, but the environmentMap texture is by default in LinearEncoding
  //to fix this we have just to go where we load our environment maps and tell that the encoding is a sRGBEncoding 
  // Go up to find to 2*

  //FInally we should remember to apply the sRGBEncoding for all those texture that we see 
  //and especially, we should remember to apply LinearEncoding to all those texture
  // that we cannot see directly, but add details to the texture we see (like the normalMap ecc) .
  //And this is because sRGBEncoding "reduce" the value in relation to what we can "perceive". 
  //Linear Encoding, instead, mantain the value, so we need this encoding for the special texture

  //Fortunately GLTFLoader has implemented this rule, so all the texture loaded from it
  //will have the right encoding automatically 


  //We now have a great base , but still isn't realistic at all, so let's go to see
  //how to improve this with tone mapping

  
  
  ////////////////////////////// TONE MAPPING  ///////////////////////////////////////
  //Tone mapping intends to convert High Dynamic Range (HDR) values to Low Dynamic Range (LDR) value
  //In this case we don't have HDR texture in our scene so we are not experiencing the main purpouse of the tone mapping,
  //but in can be used in our also in our scene, and even if our are LDR , it will be better
  
  //We can change the tone mapping property with this values:
  // THREE.NoToneMapping (default)
  // THREE.LinearToneMapping
  // THREE.ReinhardToneMapping
  // THREE.CineonToneMapping
  // THREE.ACESFilmicToneMapping

  renderer.toneMapping = THREE.ReinhardToneMapping

  //we can also change the toneMapping exposure
  //you can see that like how much light we let in,  and the algorithm wil handle it its way
  
  renderer.toneMappingExposure = 3




  //Let's add the toneMapping to dat.GUI , to be able to chose the value we want and test the different one 
  //Note that we use the third parameters of dat.GUI, to pass in an object that it's used to create a dropdown select for choose from the different values (the object property)
  gui.add(renderer, "toneMapping", {
    Not: THREE.NoToneMapping,
    Linear: THREE.LinearToneMapping,
    Reinhard: THREE.ReinhardToneMapping,
    Cineon: THREE.CineonToneMapping,
    ACESFilmic: THREE.ACESFilmicToneMapping
  }).onChange(()=>{
    renderer.toneMapping = Number(renderer.toneMapping)
    updateAllMaterials()
  })


  //add the toneMappingExposure too to dat.GUI
  gui.add(renderer, "toneMappingExposure").min(0).max(10).step(0.001)



  ///////////////////////////////// ANTIALIASING //////////////////
  //aliasing appear when the render has to decide if the "edges", i confini, 
  //of an object is on a pixel or not, and there are times in which the geometry 
  //slightly pass on a pixel and the render has to decide if draw it or not, 
  //and that's when the effect call aliasing appear 
  //("it's like the "stair" effect you see on light and shadow on a low definition videogames or when you play with low settings)
 
  //One easy solution wuold be increase our render resolution to the double, and each pixel color will automatically be averaged from the 4 pixels renderd
  //This is called super sampling (SSAA) or fullscreen sampling (FSAA), it's easy and efficient but not performant (this "divide" by 4 , every pixel of the scene)


  //Another solution called multi sampling (MSAA), will also render multiple values per pixel (usually 4) like for the super sampling but only on the geometries edges
  // values of the pixel are then averaged to get the final pixel value (this devide by 4 only the pixel on the edges o the objects, not all the surface, so it's much performant)
  
  //most recent GPU can handles this multi sampling antialiasing, and in threejs can be setup in a simple way
  //just change the antialias property to true during the istantiation of the renderer (NOT AFTER)

  //NOTES that the screen with a pixel ratio above 1 don't really need antialias
  //the best solution would be to activate the antialias only for screen with a pixel ratio below 2 

  

  ////////////////////////////////////// SHADOWS ///////////////////////////////////
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap



  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true; // permette uno effetto smooth una volta che rilasciamo l'elemento, rallenta fino a fermarsi
});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}
</style>