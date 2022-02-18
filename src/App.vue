<template>
    <Lunchbox
        :cameraPosition="[10, -15, 10]"
        :cameraLook="[0, 0, 0]"
        background="black"
    >
        <pointLight />
        <pointLight :intensity="0.2" color="blue" :position-z="25" />

        <instancedMesh
            :args="['$attached.geometry', '$attached.material', count]"
            @added="onAdded"
            :rotation-y="rotation * -0.4"
        >
            <octahedronGeometry />
            <meshStandardMaterial color="skyblue" />
        </instancedMesh>

        <!-- sun -->
        <mesh>
            <sphereGeometry />
            <meshBasicMaterial color="antiquewhite" />
        </mesh>

        <!-- miscs -->
        <mesh
            :rotation="[rotation, rotation, rotation]"
            :scale="0.2"
            :position-y="(Math.sin(rotation) * 0.4 + 0.5) * 10"
        >
            <octahedronGeometry />
            <meshStandardMaterial color="skyblue" />
        </mesh>
        <mesh
            :rotation="[rotation, rotation, rotation]"
            :scale="0.2"
            :position-y="(Math.sin(rotation) * 0.4 + 0.5) * -10"
        >
            <octahedronGeometry />
            <meshStandardMaterial color="skyblue" />
        </mesh>
    </Lunchbox>
</template>

<script setup lang="ts">
import { ref, toRaw } from 'vue'
import {
    Lunch,
    onBeforeRender,
    onStart,
    setCustomRender,
    useScene,
} from 'lunchboxjs'
import * as THREE from 'three'

const blank = new THREE.Object3D()
blank.scale.setScalar(0.2)
const blankV3 = new THREE.Vector3()

const coords = Array.from(
    new THREE.SphereGeometry(10, 30, 30, 10).attributes.position.array
)
const points: [number, number, number][] = []
for (let i = 0; i < coords.length; i += 3) {
    const v: [number, number, number] = [
        coords[i],
        coords[i + 1],
        coords[i + 2],
    ]
    if (!points.find((n) => n[0] === v[0] && n[1] === v[1] && n[2] === v[2])) {
        points.push(v)
    }
}

const count = points.length
let iMesh: THREE.InstancedMesh

const onAdded = ({ instance }: { instance: THREE.InstancedMesh }) => {
    iMesh = instance
    iMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage)
}

const rotation = ref(0)

onBeforeRender(() => {
    if (!iMesh) return

    const now = Date.now() * 0.0001
    rotation.value = now

    for (let i = 0; i < count; i++) {
        blank.rotation.z = Math.sin(now * 10 + i)
        blank.position.set(...points[i])
        const nextPoint = points[(i + 1) % points.length]
        blankV3.set(...nextPoint)
        blank.position.lerp(blankV3, Math.sin(now))
        blank.updateMatrix()
        iMesh.setMatrixAt(i, blank.matrix)
    }

    iMesh.instanceMatrix.needsUpdate = true
})

// postprocessing
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass'

let composer: EffectComposer | null = null

setCustomRender((opts) => {
    const canvas = opts.renderer?.domElement

    // ignore if no canvas
    if (!canvas || !opts.scene || !opts.camera) return

    // setup effect composer if needed
    if (!composer) {
        composer = new EffectComposer(opts.renderer as THREE.WebGLRenderer)

        const renderPass = new RenderPass(opts.scene, opts.camera)
        composer.addPass(renderPass)

        const bloomPass = new UnrealBloomPass(
            new THREE.Vector2(canvas.width, canvas.height),
            0.85,
            0.1,
            0.4
        )
        composer.addPass(bloomPass)
    }

    composer.render()
})
</script>