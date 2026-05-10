<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader.js'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const contenedor3D = ref(null)

let escena, camara, renderizador, controles, animacionId
let raycaster, raton

const arregloCasillas = []
const piezas = []
let piezaSeleccionada = null

onMounted(() => {
  // --- ESCENA ---
  escena = new THREE.Scene()
  escena.background = new THREE.Color(0x050505)

  // --- CÁMARA ---
  camara = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000)
  camara.position.set(0, 35, 40)

  // --- RENDERIZADOR ---
  renderizador = new THREE.WebGLRenderer({ antialias: true })
  renderizador.setSize(window.innerWidth, window.innerHeight)
  renderizador.shadowMap.enabled = true
  contenedor3D.value.appendChild(renderizador.domElement)

  // --- LUCES ---
  const luzAmbiental = new THREE.AmbientLight(0xffffff, 0.8)
  escena.add(luzAmbiental)

  const luzDireccional = new THREE.DirectionalLight(0xffffff, 1.5)
  luzDireccional.position.set(20, 50, 20)
  luzDireccional.castShadow = true
  escena.add(luzDireccional)

  // --- GENERACIÓN DEL TABLERO ---
  const tamañoCasilla = 5
  const geometriaCasilla = new THREE.BoxGeometry(tamañoCasilla, 1, tamañoCasilla)
  const materialBlanco = new THREE.MeshStandardMaterial({ color: 0xeeeeee })
  const materialNegro = new THREE.MeshStandardMaterial({ color: 0x222222 })

  for (let fila = 0; fila < 8; fila++) {
    for (let col = 0; col < 8; col++) {
      const esBlanca = (fila + col) % 2 === 0
      const casilla = new THREE.Mesh(geometriaCasilla, esBlanca ? materialBlanco : materialNegro)
      casilla.receiveShadow = true
      casilla.position.set((col * tamañoCasilla) - 17.5, 0, (fila * tamañoCasilla) - 17.5)
      escena.add(casilla)
      arregloCasillas.push(casilla)
    }
  }

  // --- CARGA DE PIEZAS (Lógica de colocación) ---
  const setupInicial = () => {
    // Blancas (Fila 0 y 1)
    const orden = ['/torre.stl', '/caballo.stl', '/alfiere.stl', '/regina.stl', '/re.stl', '/alfiere.stl', '/caballo.stl', '/torre.stl']
    orden.forEach((ruta, i) => cargarPieza(ruta, arregloCasillas[i].position.x, arregloCasillas[i].position.z, 0xffffff))
    for (let i = 8; i <= 15; i++) cargarPieza('/pedone.stl', arregloCasillas[i].position.x, arregloCasillas[i].position.z, 0xffffff)

    // Negras (Fila 6 y 7) - Color Morado como tenías
    orden.forEach((ruta, i) => cargarPieza(ruta, arregloCasillas[i + 56].position.x, arregloCasillas[i + 56].position.z, 0x8000ff))
    for (let i = 48; i <= 55; i++) cargarPieza('/pedone.stl', arregloCasillas[i].position.x, arregloCasillas[i].position.z, 0x8000ff)
  }

  setupInicial()

  // --- INTERACCIÓN ---
  raycaster = new THREE.Raycaster()
  raton = new THREE.Vector2()
  renderizador.domElement.addEventListener('pointerdown', alHacerClic)

  // --- CONTROLES ---
  controles = new OrbitControls(camara, renderizador.domElement)
  controles.enableDamping = true

  // --- BUCLE DE ANIMACIÓN ---
  const animar = () => {
    animacionId = requestAnimationFrame(animar)
    controles.update()
    renderizador.render(escena, camara)
  }
  animar()

  window.addEventListener('resize', actualizarVentana)
})

// --- FUNCIONES DE APOYO ---
function cargarPieza(ruta, x, z, color) {
  const cargador = new STLLoader()
  cargador.load(ruta, (geometria) => {
    geometria.center()
    const material = new THREE.MeshStandardMaterial({ color, metalness: 0.4, roughness: 0.5 })
    const pieza = new THREE.Mesh(geometria, material)

    geometria.computeBoundingBox()
    const size = new THREE.Vector3()
    geometria.boundingBox.getSize(size)
    const escala = 6 / Math.max(size.x, size.y, size.z)
    
    pieza.scale.setScalar(escala)
    pieza.rotation.x = -Math.PI / 2
    pieza.position.set(x, 3, z) // Elevadas un poco sobre el tablero
    pieza.castShadow = true

    escena.add(pieza)
    piezas.push(pieza)
  })
}

function alHacerClic(evento) {
  const rect = renderizador.domElement.getBoundingClientRect()
  raton.x = ((evento.clientX - rect.left) / rect.width) * 2 - 1
  raton.y = -((evento.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(raton, camara)

  const interseccionesPiezas = raycaster.intersectObjects(piezas)
  if (interseccionesPiezas.length > 0) {
    // Desmarcar anterior si existiera (opcional: feedback visual aquí)
    piezaSeleccionada = interseccionesPiezas[0].object
  }

  const interseccionesCasillas = raycaster.intersectObjects(arregloCasillas)
  if (interseccionesCasillas.length > 0 && piezaSeleccionada) {
    const casilla = interseccionesCasillas[0].object
    piezaSeleccionada.position.x = casilla.position.x
    piezaSeleccionada.position.z = casilla.position.z
    // Soltamos la pieza tras moverla
    piezaSeleccionada = null 
  }
}

function actualizarVentana() {
  camara.aspect = window.innerWidth / window.innerHeight
  camara.updateProjectionMatrix()
  renderizador.setSize(window.innerWidth, window.innerHeight)
}

onBeforeUnmount(() => {
  cancelAnimationFrame(animacionId)
  window.removeEventListener('resize', actualizarVentana)
  if (renderizador) {
    renderizador.domElement.removeEventListener('pointerdown', alHacerClic)
    renderizador.dispose()
  }
})
</script>

<template>
  <div class="contenedor-principal">
    <div class="interfaz-superior">
      <h1>AJEDREZ 3D</h1>
      <div class="decoracion-linea"></div>
      <p class="clase-info">Sistemas SSR | Tarea 2</p>
    </div>

    <div ref="contenedor3D" class="visor"></div>

    <div class="instrucciones-flotantes">
      <p>Selecciona una pieza y toca una casilla para mover.</p>
    </div>
  </div>
</template>

<style scoped>
.contenedor-principal {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: #050505;
  color: white;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.visor {
  width: 100%;
  height: 100%;
}

/* ESTILO DEL TÍTULO SUPERIOR */
.interfaz-superior {
  position: absolute;
  top: 40px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  text-align: center;
  pointer-events: none; /* No bloquea el mouse para el 3D */
}

.interfaz-superior h1 {
  margin: 0;
  font-size: 3.5rem;
  font-weight: 900;
  letter-spacing: 12px;
  text-shadow: 0 0 20px rgba(128, 0, 255, 0.8);
  background: linear-gradient(to bottom, #ffffff, #a855f7);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.decoracion-linea {
  height: 2px;
  width: 100px;
  background: #8000ff;
  margin: 10px auto;
  box-shadow: 0 0 10px #8000ff;
}

.clase-info {
  font-size: 0.8rem;
  letter-spacing: 3px;
  text-transform: uppercase;
  opacity: 0.6;
}

/* ESTILO DE LAS INSTRUCCIONES */
.instrucciones-flotantes {
  position: absolute;
  bottom: 40px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  padding: 15px 30px;
  border-radius: 50px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  z-index: 100;
}

.instrucciones-flotantes p {
  margin: 0;
  font-size: 0.9rem;
  font-weight: 300;
  color: #e2e8f0;
}
</style>