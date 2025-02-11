<template>
<svg viewBox="0 0 248 500"
     preserveAspectRatio="xMidYMid meet"
     ref="mmuSpoolSvg">

    <defs>
        <path id="oval" d="M 0 -63 C 35 -63 63 -35 63 0 C 63 35 35 63 0 63 C -35 63 -63 35 -63 0 C -63 -35 -35 -63 0 -63 z" vector-effect="non-scaling-stroke"/>
        <path id="center" d="M 0 -63 C 35 -63 63 -35 63 0 C 63 35 35 63 0 63 L -624 63 L -624 -63 z" vector-effect="non-scaling-stroke"/>
    </defs>
    <filter id="blur_wheel2" width="1.3" height="1.16">
        <feGaussianBlur in="SourceAlpha" stdDeviation="3"/>
        <feOffset dx="18" dy="0" result="oBlur"/>
        <feFlood flood-color="#000" flood-opacity=".67"/>
        <feComposite in2="oBlur" operator="in"/>
        <feMerge>
            <feMergeNode/>
            <feMergeNode in="SourceGraphic"/>
        </feMerge>
    </filter>
    <g transform="matrix(0.59,0,0,3.95,197,250)">
        <use href="#oval" style="filter:url(#blur_wheel2)" :fill="spoolWheelColor"/>
        <use href="#oval" transform="scale(0.41)" style="filter:url(#blur_wheel2)" :fill="spoolWheelColor"/>
        <use href="#center" transform="scale(0.41)" :fill="spoolWheelColor"/>
    </g>
    <path v-if="filamentAmount > 0" ref="filament"
          d="M 0 -63 C 35 -63 63 -35 63 0 C 63 35 35 63 0 63 L -424 63 L -424 -63 z" vector-effect="non-scaling-stroke"
          :fill="gateColor"
          :transform="'matrix(' + computedScale(0.28, 0.4) + ',0,0,' + computedScale(1.65, 3.5) + ',197,250)'"/>
    <g transform="matrix(0.59,0,0,3.95,37,250)">
        <use href="#oval" style="filter:url(#blur_wheel2)" :fill="spoolWheelColor"/>
        <use href="#oval" transform="scale(0.41)" style="fill:#111111"/>
    </g>
    <text v-if="filamentAmount > 0 && filamentAmount < 100" x="152" y="270" text-anchor="middle"
          font-weight="bold" font-size="56px" :fill="contrastColor">
        {{ filamentAmount }}%
    </text>
</svg>
</template>

<script lang="ts">
import Component from 'vue-class-component'
import { Mixins, Prop } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import MmuMixin from '@/components/mixins/mmu'
import { ServerSpoolmanStateSpool } from '@/store/server/spoolman/types'

@Component({ })
export default class MmuSpool extends Mixins(BaseMixin, MmuMixin) {
    @Prop({ required: false, default: null }) readonly thisGate: number | null
    @Prop({ required: false, default: "#AD8762" }) readonly spoolWheelColor: string

    contrastColor: string = "black"

    computedScale(start, end) {
        return start + (end - start) * (this.filamentAmount / 100)
    }

    get filamentAmount(): number {
        if (this.thisGate === null) {
            return 100
        }
        const gateStatus = this.$store.state.printer.mmu.gate_status[this.thisGate]
        if (gateStatus === 0) {
            return 0
        }
        const spoolmanSupport = this.$store.state.printer.mmu.spoolman_support
        const gateSpoolId = this.$store.state.printer.mmu.gate_spool_id[this.thisGate]
        const spools = this.$store.state.server.spoolman?.spools ?? []
        const spoolmanSpool = spools.find((spool: ServerSpoolmanStateSpool) => spool.id === gateSpoolId) ?? null
        let amount = 100
        if (gateSpoolId > 0 && spoolmanSupport !== "off") {
            // Pull live from spoolman and calculate percentage
            let remaining = spoolmanSpool?.remaining_weight ?? null
            let total = spoolmanSpool?.filament?.weight ?? null
            if (remaining !== null && total !== null) {
                amount = Math.round(Math.max(0, Math.min(100, (remaining / total) * 100)))
            }
        }
        return amount
    }

    get gateColor(): string {
        // Need spool to be rendered first
        this.$nextTick(() => {
            this.computeContrastColor()
        })

        if (this.thisGate === null) {
            return "#808080E0"
        }
        // Happy Hare syncs with spoolman so believe gate map
        const gateColor = this.$store.state.printer.mmu.gate_color[this.thisGate]
        const hexColorPattern = /^[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$/
        if (hexColorPattern.test(gateColor) && !gateColor.startsWith('#')) {
            return '#' + gateColor
        }
        return gateColor
    }

    mounted() {
        this.computeContrastColor()
    }

    computeContrastColor(): void {
        if (!this.$refs.filament) {
            this.contrastColor = "black"
            return
        }

        const fillColor = window.getComputedStyle(this.$refs.filament).fill
        const rgbaMatch = fillColor.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)(?:,\s*([\d.]+))?\)/)
    
        if (rgbaMatch) {
            const [r, g, b] = rgbaMatch.slice(1, 4).map(Number)
            const luminance = this.getLuminance({ r, g, b })
            this.contrastColor = luminance > 0.5 ? 'black' : 'white'
        } else {
            this.contrastColor = "black"
        }
    }

    getLuminance({ r, g, b }) {
        const a = [r, g, b].map(function (v) {
            v /= 255
            return v <= 0.03928 ? v / 12.92 : Math.pow(((v + 0.055) / 1.055), 2.4)
        })
        return a[0] * 0.2126 + a[1] * 0.7152 + a[2] * 0.0722
    }
}
</script>

<style scoped>
</style>
