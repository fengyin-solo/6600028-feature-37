<script setup lang="ts">
import { ref, computed } from 'vue'
import { useFluidStore } from '../store/fluid'
import { PRESETS } from '../utils/sph-engine'
import type { Preset, SimParams } from '../types'

const store = useFluidStore()

const expandedGroups = ref<Record<string, boolean>>({
  physics: true,
  simulation: true,
  particles: true,
})

interface ParamConfig {
  key: keyof SimParams | 'particleCount'
  label: string
  group: 'physics' | 'simulation' | 'particles'
  min: number
  max: number
  step: number
  format: (v: number) => string
  important?: boolean
  tip?: string
  requireReset?: boolean
}

const paramConfigs: ParamConfig[] = [
  {
    key: 'gravity',
    label: '重力',
    group: 'physics',
    min: 0,
    max: 30,
    step: 0.1,
    format: (v) => v.toFixed(1),
    important: true,
    tip: '控制粒子下落的加速度，值越大下落越快',
  },
  {
    key: 'viscosity',
    label: '粘性',
    group: 'physics',
    min: 0,
    max: 5,
    step: 0.1,
    format: (v) => v.toFixed(1),
    important: true,
    tip: '流体的粘稠程度，值越高流体越"稠"，流动越慢',
  },
  {
    key: 'restDensity',
    label: '静止密度',
    group: 'physics',
    min: 500,
    max: 2000,
    step: 50,
    format: (v) => v.toFixed(0),
    tip: '流体在静止状态下的参考密度，影响压力计算',
  },
  {
    key: 'gasConstant',
    label: '气体常数',
    group: 'physics',
    min: 500,
    max: 5000,
    step: 100,
    format: (v) => v.toFixed(0),
    tip: '控制压力对密度变化的敏感度，值越大流体越"硬"',
  },
  {
    key: 'smoothingRadius',
    label: '光滑半径',
    group: 'simulation',
    min: 8,
    max: 50,
    step: 1,
    format: (v) => v.toFixed(0),
    important: true,
    tip: '粒子间相互作用的影响范围，值越大粒子间作用越远',
  },
  {
    key: 'dt',
    label: '时间步长',
    group: 'simulation',
    min: 0.001,
    max: 0.02,
    step: 0.001,
    format: (v) => v.toFixed(4),
    tip: '每帧模拟的时间间隔，值越大模拟越快但可能不稳定',
  },
  {
    key: 'damping',
    label: '边界阻尼',
    group: 'simulation',
    min: 0,
    max: 1,
    step: 0.05,
    format: (v) => v.toFixed(2),
    tip: '粒子碰到边界时的能量衰减，0 完全弹回，1 完全不弹',
  },
  {
    key: 'boundaryStiffness',
    label: '边界刚度',
    group: 'simulation',
    min: 1000,
    max: 30000,
    step: 1000,
    format: (v) => v.toFixed(0),
    tip: '边界排斥力的强度，值越大粒子越难穿过边界',
  },
  {
    key: 'particleMass',
    label: '粒子质量',
    group: 'particles',
    min: 0.5,
    max: 10,
    step: 0.5,
    format: (v) => v.toFixed(1),
    tip: '每个粒子的质量，影响密度和压力计算',
  },
  {
    key: 'particleCount',
    label: '粒子数量',
    group: 'particles',
    min: 200,
    max: 2000,
    step: 50,
    format: (v) => v.toFixed(0),
    important: true,
    tip: '模拟的粒子总数，越多效果越细腻但性能消耗越大',
    requireReset: true,
  },
]

const groupConfigs = [
  { key: 'physics', label: '物理属性', icon: '⚡' },
  { key: 'simulation', label: '模拟设置', icon: '⚙️' },
  { key: 'particles', label: '粒子参数', icon: '•' },
] as const

const groupedParams = computed(() => {
  const groups: Record<string, ParamConfig[]> = {}
  for (const config of paramConfigs) {
    if (!groups[config.group]) {
      groups[config.group] = []
    }
    groups[config.group].push(config)
  }
  return groups
})

function selectPreset(preset: Preset) {
  store.initSimulation(preset)
}

function toggleRun() {
  if (store.isRunning) {
    store.stop()
  } else {
    store.start()
  }
}

function reset() {
  store.reset()
}

function stepOnce() {
  store.stepOnce()
}

function toggleGroup(groupKey: string) {
  expandedGroups.value[groupKey] = !expandedGroups.value[groupKey]
}

function onParamInput(e: Event, key: keyof SimParams | 'particleCount') {
  const value = parseFloat((e.target as HTMLInputElement).value)
  if (key === 'particleCount') {
    store.particleCount = value
  } else {
    store.updateParam(key, value)
  }
}

function getParamValue(key: keyof SimParams | 'particleCount'): number {
  if (key === 'particleCount') {
    return store.particleCount
  }
  return store.params[key]
}

function isModified(key: keyof SimParams | 'particleCount'): boolean {
  if (key === 'particleCount') {
    return store.isParticleCountModified
  }
  return store.isParamModified(key)
}

function resetParam(key: keyof SimParams | 'particleCount') {
  if (key === 'particleCount') {
    store.resetParticleCount()
  } else {
    store.resetParam(key)
  }
}

function resetAllParams() {
  store.resetAllParams()
}

const hasAnyModified = computed(() => {
  return paramConfigs.some((c) => isModified(c.key))
})
</script>

<template>
  <div class="w-72 bg-gray-800 rounded-lg border border-gray-700 p-4 flex flex-col gap-4 overflow-auto h-full">
    <!-- Presets -->
    <div>
      <h3 class="text-xs font-semibold text-gray-400 uppercase tracking-wider mb-2">预设场景</h3>
      <div class="grid grid-cols-2 gap-2">
        <button
          v-for="preset in PRESETS"
          :key="preset.name"
          @click="selectPreset(preset)"
          class="text-xs px-2 py-2 rounded transition text-left"
          :class="store.currentPreset.name === preset.name
            ? 'bg-blue-600 text-white'
            : 'bg-gray-700 text-gray-300 hover:bg-gray-600'"
        >
          {{ preset.label }}
        </button>
      </div>
      <p class="text-xs text-gray-500 mt-1">{{ store.currentPreset.description }}</p>
    </div>

    <!-- Controls -->
    <div class="flex gap-2">
      <button
        @click="toggleRun"
        class="flex-1 py-2 rounded text-sm font-medium transition"
        :class="store.isRunning
          ? 'bg-red-600 hover:bg-red-700 text-white'
          : 'bg-green-600 hover:bg-green-700 text-white'"
      >
        {{ store.isRunning ? '暂停' : '开始' }}
      </button>
      <button
        @click="reset"
        class="flex-1 bg-gray-700 hover:bg-gray-600 text-gray-200 py-2 rounded text-sm transition"
      >
        重置
      </button>
      <button
        @click="stepOnce"
        :disabled="store.isRunning"
        class="flex-1 bg-gray-700 hover:bg-gray-600 disabled:opacity-40 text-gray-200 py-2 rounded text-sm transition"
      >
        单步
      </button>
    </div>

    <!-- Parameter Groups -->
    <div class="space-y-2 flex-1 overflow-auto">
      <div class="flex items-center justify-between mb-1">
        <h3 class="text-xs font-semibold text-gray-400 uppercase tracking-wider">模拟参数</h3>
        <button
          v-if="hasAnyModified"
          @click="resetAllParams"
          class="text-xs text-blue-400 hover:text-blue-300 transition"
        >
          全部重置
        </button>
      </div>

      <div
        v-for="group in groupConfigs"
        :key="group.key"
        class="bg-gray-900 rounded-md overflow-hidden"
      >
        <button
          @click="toggleGroup(group.key)"
          class="w-full px-3 py-2 flex items-center justify-between text-left hover:bg-gray-700 transition"
        >
          <div class="flex items-center gap-2">
            <span class="text-sm">{{ group.icon }}</span>
            <span class="text-sm font-medium text-gray-200">{{ group.label }}</span>
            <span
              v-if="groupedParams[group.key]?.some(c => isModified(c.key))"
              class="w-2 h-2 rounded-full bg-amber-400"
              title="有已修改的参数"
            />
          </div>
          <span
            class="text-gray-500 text-xs transition-transform"
            :class="{ 'rotate-90': expandedGroups[group.key] }"
          >
            ▶
          </span>
        </button>

        <div
          v-show="expandedGroups[group.key]"
          class="px-3 pb-3 space-y-3"
        >
          <div
            v-for="config in groupedParams[group.key]"
            :key="config.key"
            class="relative"
            :class="{ 'pt-2': config.important }"
          >
            <div
              v-if="config.important"
              class="absolute -top-0.5 left-0 right-0 h-0.5 bg-gradient-to-r from-amber-500/50 via-amber-400 to-amber-500/50 rounded-t"
            />
            <div class="flex items-start justify-between mb-1">
              <label class="flex items-center gap-1 text-xs text-gray-400">
                <span :class="{ 'text-amber-400 font-medium': config.important }">
                  {{ config.label }}
                </span>
                <span
                  v-if="config.important"
                  class="text-amber-500 text-[10px] px-1 py-0.5 bg-amber-500/10 rounded"
                >
                  重点
                </span>
                <span
                  v-if="config.tip"
                  class="group/tip relative cursor-help"
                >
                  <span class="text-gray-600 hover:text-gray-400">ⓘ</span>
                  <span class="absolute bottom-full left-1/2 -translate-x-1/2 mb-1 w-40 p-2 bg-gray-900 border border-gray-600 rounded text-[11px] text-gray-300 opacity-0 invisible group-hover/tip:opacity-100 group-hover/tip:visible transition-all z-10 pointer-events-none">
                    {{ config.tip }}
                  </span>
                </span>
              </label>
              <div class="flex items-center gap-1">
                <span class="text-xs text-gray-300 font-mono">{{ config.format(getParamValue(config.key)) }}</span>
                <button
                  v-if="isModified(config.key)"
                  @click="resetParam(config.key)"
                  class="text-[10px] text-blue-400 hover:text-blue-300 transition px-1 rounded hover:bg-blue-500/10"
                  title="恢复默认值"
                >
                  ↺
                </button>
              </div>
            </div>
            <input
              type="range"
              :min="config.min"
              :max="config.max"
              :step="config.step"
              :value="getParamValue(config.key)"
              @input="onParamInput($event, config.key)"
              class="w-full h-1.5"
              :class="isModified(config.key) ? 'accent-amber-500' : 'accent-blue-500'"
            />
            <p
              v-if="config.requireReset"
              class="text-[10px] text-gray-600 mt-0.5"
            >
              重置后生效
            </p>
          </div>
        </div>
      </div>
    </div>

    <!-- Stats -->
    <div class="pt-3 border-t border-gray-700">
      <h3 class="text-xs font-semibold text-gray-400 uppercase tracking-wider mb-2">运行状态</h3>
      <div class="grid grid-cols-2 gap-2 text-xs">
        <div class="bg-gray-900 rounded px-2 py-1.5">
          <span class="text-gray-500">FPS</span>
          <p class="text-green-400 font-mono text-sm">{{ store.fps }}</p>
        </div>
        <div class="bg-gray-900 rounded px-2 py-1.5">
          <span class="text-gray-500">粒子数</span>
          <p class="text-blue-400 font-mono text-sm">{{ store.particleArray.length }}</p>
        </div>
        <div class="bg-gray-900 rounded px-2 py-1.5">
          <span class="text-gray-500">平均密度</span>
          <p class="text-yellow-400 font-mono text-sm">{{ store.avgDensity.toFixed(0) }}</p>
        </div>
        <div class="bg-gray-900 rounded px-2 py-1.5">
          <span class="text-gray-500">最大速度</span>
          <p class="text-red-400 font-mono text-sm">{{ store.maxVelocity.toFixed(1) }}</p>
        </div>
      </div>
    </div>
  </div>
</template>
