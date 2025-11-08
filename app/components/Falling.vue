<!-- comps/Falling.vue -->
<script setup lang="ts">
import { BasicShadowMap, SRGBColorSpace, NoToneMapping } from 'three'
import type { Points } from 'three'
import { OrbitControls, GLTFModel } from '@tresjs/cientos'
// Nuxt 自动引入:
// - TresCanvas (来自 @tresjs/core)
// - useLoop (来自 @tresjs/core)
// - OrbitControls, GLTFModel (来自 @tresjs/cientos)
// - ref (来自 Vue)

// 创建星空粒子数据
const particlesCount = 10000
const radius = 100

// 生成粒子位置和颜色
const positions = new Float32Array(particlesCount * 3)
const colors = new Float32Array(particlesCount * 3)

for (let i = 0; i < particlesCount; i++) {
    const i3 = i * 3

    // 随机位置
    positions[i3] = (Math.random() - 0.5) * radius * 2      // x: -100 到 100
    positions[i3 + 1] = (Math.random() - 0.5) * radius * 2  // y: -100 到 100
    positions[i3 + 2] = (Math.random() - 0.5) * radius * 2  // z: -100 到 100

    // 随机颜色
    const colorChoice = Math.random()

    if (colorChoice < 0.7) {
        // 70% 白色星星
        colors[i3] = 1
        colors[i3 + 1] = 1
        colors[i3 + 2] = 1
    } else if (colorChoice < 0.85) {
        // 15% 蓝色星星
        colors[i3] = 0.5 + Math.random() * 0.5
        colors[i3 + 1] = 0.7 + Math.random() * 0.3
        colors[i3 + 2] = 1
    } else {
        // 15% 黄色/橙色星星
        colors[i3] = 1
        colors[i3 + 1] = 0.7 + Math.random() * 0.3
        colors[i3 + 2] = 0.5 + Math.random() * 0.5
    }
}

// 粒子系统引用 (指定类型为 Points 或 null)
const particlesRef = ref<Points | null>(null)

// 模型引用
const modelRef = ref<any>(null)

// 模型路径 - 使用 useRuntimeConfig 获取正确的 baseURL
const config = useRuntimeConfig()
const modelPath = `${config.app.baseURL}Falling.glb`.replace('//', '/')

// 修复模型材质的函数
const fixModelMaterials = (model: any) => {
    console.log('🔧 开始修复模型材质', model)

    let meshCount = 0
    model.traverse((child: any) => {
        console.log('🔍 遍历子对象:', {
            name: child.name,
            type: child.type,
            isMesh: child.isMesh,
            constructor: child.constructor.name
        })

        if (child.isMesh || child.type === 'Mesh') {
            meshCount++
            const material = child.material
            console.log('✅ 找到网格:', child.name, '材质类型:', material.type)

            // ===== MC 风格贴图透明度处理 =====
            material.transparent = true
            material.alphaTest = 0.5
            material.depthWrite = true
            material.depthTest = true
            material.side = 2  // THREE.DoubleSide

            // 如果有贴图
            if (material.map) {
                material.map.needsUpdate = true
                material.map.colorSpace = SRGBColorSpace
                console.log('📷 找到贴图')
            }

            material.needsUpdate = true
            console.log('✅ 材质已修复:', child.name)
        }
    })

    console.log(`📊 总共找到 ${meshCount} 个网格`)
}

// 动画循环 (新版本 API: useLoop 替代 useRenderLoop)
// 必须在组件挂载后才能访问 TresCanvas 上下文
onMounted(() => {
    console.log('🎬 组件已挂载，启动动画循环')

    // 监听模型加载 - 使用轮询检查模型是否真正加载完成
    watch(modelRef, (newModel) => {
        if (newModel) {
            console.log('✅ 检测到模型引用:', newModel)

            // 轮询检查模型是否包含网格
            const checkModelLoaded = () => {
                const model = newModel.scene || newModel.model || newModel.$el || newModel

                let hasMesh = false
                if (model && typeof model.traverse === 'function') {
                    model.traverse((child: any) => {
                        if (child.isMesh || child.type === 'Mesh') {
                            hasMesh = true
                        }
                    })
                }

                if (hasMesh) {
                    console.log('🎉 模型网格已加载完成!')
                    fixModelMaterials(model)
                } else {
                    console.log('⏳ 模型还在加载中，500ms后重试...')
                    setTimeout(checkModelLoaded, 500)
                }
            }

            // 延迟一点再开始检查
            setTimeout(checkModelLoaded, 100)
        }
    }, { immediate: true })

    const { onBeforeRender } = useLoop()

    onBeforeRender(({ elapsed }: { elapsed: number }) => {
        if (particlesRef.value) {
            // 缓慢旋转粒子系统
            particlesRef.value.rotation.y += 0.0002
            particlesRef.value.rotation.x += 0.0001

            // 粒子闪烁效果
            const time = elapsed
            const positionAttr = particlesRef.value.geometry.attributes.position

            // 类型保护: 确保 position 属性存在
            if (positionAttr?.array) {
                const posArray = positionAttr.array as Float32Array

                // 让一些粒子轻微移动(模拟闪烁)
                for (let i = 0; i < posArray.length - 1; i += 30) {
                    // TypeScript 类型断言: 数组索引在循环内是安全的
                    posArray[i]! += Math.sin(time + i) * 0.001
                    posArray[i + 1]! += Math.cos(time + i) * 0.001
                }

                // 标记位置需要更新
                positionAttr.needsUpdate = true
            }
        }
    })
})

console.log(`✨ 创建了 ${particlesCount} 个星星粒子`)
console.log('🎯 相机距离限制: 最近5, 最远50')
console.log('💡 夜空光照已添加')
console.log('🔍 准备加载模型: /Falling.glb')
</script>

<template>
    <!-- 相机 -->
    <TresPerspectiveCamera :position="[0, 5, 20]" :fov="75" :near="0.1" :far="1000" />

    <!-- 轨道控制器 -->
    <OrbitControls :enable-damping="true" :damping-factor="0.05" :auto-rotate="true" :auto-rotate-speed="0.3"
        :min-distance="5" :max-distance="50" />

    <!-- ===== 光照系统 ===== -->
    <!-- 环境光 - 提供整体基础照明 -->
    <TresAmbientLight color="#4466ff" :intensity="1" />

    <!-- 主光源 - 模拟月光 -->
    <TresDirectionalLight color="#aaccff" :intensity="1" :position="[5, 10, 5]" />

    <!-- 补光 - 从侧面补光 -->
    <TresDirectionalLight color="#6688ff" :intensity="0.3" :position="[-5, 3, -5]" />

    <!-- 底部补光 -->
    <TresDirectionalLight color="#8899ff" :intensity="0.2" :position="[0, -5, 0]" />

    <!-- ===== 星空粒子系统 ===== -->
    <TresPoints ref="particlesRef">
        <TresBufferGeometry>
            <TresBufferAttribute :args="[positions, 3]" attach="attributes-position" />
            <TresBufferAttribute :args="[colors, 3]" attach="attributes-color" />
        </TresBufferGeometry>
        <TresPointsMaterial :size="0.15" :size-attenuation="true" :vertex-colors="true" :transparent="true"
            :opacity="0.8" :blending="2" :depth-write="false" />
    </TresPoints>

    <!-- ===== MC小人模型 ===== -->
    <GLTFModel ref="modelRef" :path="modelPath" :position="[0, 0, 0]" :scale="[2, 2, 2]" />
</template>