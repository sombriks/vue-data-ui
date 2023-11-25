<script setup>
import { ref, computed } from "vue";
import { treeShake, shiftHue, opacity, convertConfigColors, palette, convertColorToHex } from "../lib";
import mainConfig from "../default_configs.json";
import { useNestedProp } from "../useNestedProp";

const props = defineProps({
    config: {
        type: Object,
        default() {
            return {}
        }
    },
    dataset: {
        type: Array,
        default() {
            return [];
        }
    }
});

const uid = ref(`vue-ui-sparkhistogram-${Math.random()}`);
const defaultConfig = ref(mainConfig.vue_ui_sparkhistogram);

const histoConfig = computed(() => {
    return useNestedProp({
        userConfig: props.config,
        defaultConfig: defaultConfig.value
    });
});

const drawingArea = computed(() => {
    const height = histoConfig.value.style.layout.height;
    const width = histoConfig.value.style.layout.width;
    const top = histoConfig.value.style.layout.padding.top;
    const bottom = height - histoConfig.value.style.layout.padding.bottom;
    const left = histoConfig.value.style.layout.padding.left;
    const right = width - histoConfig.value.style.layout.padding.right;
    const centerY = top + ((height - top - histoConfig.value.style.layout.padding.bottom) / 2);
    const drawingHeight = height - histoConfig.value.style.layout.padding.top - histoConfig.value.style.layout.padding.bottom;
    const drawingWidth = width - histoConfig.value.style.layout.padding.left - histoConfig.value.style.layout.padding.right;
    return {
        bottom,
        centerY,
        drawingHeight,
        drawingWidth,
        height,
        left,
        right,
        top,
        width,
    }
});

const maxVal = computed(() => {
    return Math.max(...props.dataset.map(ds => Math.abs(ds.value)))
});

function toMax(val) {
    return Math.abs(val) / maxVal.value;
}

const computedDataset = computed(() => {
    return props.dataset.map((dp,i) => {
        const proportion = toMax(dp.value);
        const height = drawingArea.value.drawingHeight * proportion;
        const unitWidth = drawingArea.value.drawingWidth / props.dataset.length;
        const gap = unitWidth * (histoConfig.value.style.bars.gap / 100)
        const width = unitWidth - gap;
        const y = drawingArea.value.centerY - height / 2;
        const x = drawingArea.value.left + (gap / 2  + (i * unitWidth));
        const intensity = typeof dp.intensity === 'undefined' ? 100 : Math.round(dp.intensity * 100);
        const color = dp.value >= 0 ? `${histoConfig.value.style.bars.colors.positive}${opacity[intensity]}` : `${histoConfig.value.style.bars.colors.negative}${opacity[intensity]}`;
        const stroke = dp.value >= 0 ? histoConfig.value.style.bars.colors.positive : histoConfig.value.style.bars.colors.negative;
        const gradient = dp.value >=  0 ? `url(#gradient_positive_${i}_${uid.value})` : `url(#gradient_negative_${i}_${uid.value})`;
        const textAnchor = x + width / 2;
        return {
            ...dp,
            color,
            gradient,
            height,
            intensity,
            proportion,
            stroke,
            textAnchor,
            width,
            x,
            y
        }
    });
});

</script>

<template>
    <div :style="`width:100%;background:${histoConfig.style.backgroundColor};font-family:${histoConfig.style.fontFamily}`">
        <!-- TITLE -->
        <div v-if="histoConfig.style.title.text" :style="`width:calc(100% - 12px);background:${histoConfig.style.backgroundColor};margin:0 auto;margin:${histoConfig.style.title.margin};padding: 0 6px;text-align:${histoConfig.style.title.textAlign}`">
            <div :style="`font-size:${histoConfig.style.title.fontSize}px;color:${histoConfig.style.title.color};font-weight:${histoConfig.style.title.bold ? 'bold' : 'normal'}`">
                {{ histoConfig.style.title.text }}
            </div>
            <div v-if="histoConfig.style.title.subtitle.text" :style="`font-size:${histoConfig.style.title.subtitle.fontSize}px;color:${histoConfig.style.title.subtitle.color};font-weight:${histoConfig.style.title.subtitle.bold ? 'bold' : 'normal'}`">
                {{ histoConfig.style.title.subtitle.text }}
            </div>    
        </div>

        <svg data-cy="sparkhistogram-svg" :viewBox="`0 0 ${drawingArea.width} ${drawingArea.height}`">
            <defs>
                <radialGradient v-for="(posGrad, i) in computedDataset" :id="`gradient_positive_${i}_${uid}`"  cy="50%" cx="50%" r="50%" fx="50%" fy="50%">
                    <stop offset="0%" :stop-color="`${shiftHue(histoConfig.style.bars.colors.positive, 0.05)}${opacity[posGrad.intensity]}`"/>
                    <stop offset="100%" :stop-color="`${histoConfig.style.bars.colors.positive}${opacity[posGrad.intensity]}`"/>
                </radialGradient>

                <radialGradient v-for="(negGrad, i) in computedDataset" :id="`gradient_negative_${i}_${uid}`"  cy="50%" cx="50%" r="50%" fx="50%" fy="50%">
                    <stop offset="0%" :stop-color="`${shiftHue(histoConfig.style.bars.colors.negative, 0.05)}${opacity[negGrad.intensity]}`"/>
                    <stop offset="100%" :stop-color="`${histoConfig.style.bars.colors.negative}${opacity[negGrad.intensity]}`"/>
                </radialGradient>
            </defs>


            <rect v-for="(rect, i) in computedDataset"
                :data-cy="`sparkhistogram-rect-${i}`"
                :x="rect.x"
                :y="rect.y"
                :height="rect.height"
                :width="rect.width"
                :fill="histoConfig.style.bars.colors.gradient.show ? rect.gradient : rect.color"
                :stroke="rect.stroke"
                :stroke-width="histoConfig.style.bars.strokeWidth"
                :rx="`${histoConfig.style.bars.borderRadius * rect.proportion / 12}%`"
            />

            <text v-for="(val, i) in computedDataset"
                :data-cy="`sparkhistogram-value-label-${i}`"
                text-anchor="middle"
                :x="val.textAnchor"
                :y="val.y - histoConfig.style.labels.value.fontSize / 3"
                :font-size="histoConfig.style.labels.value.fontSize"
                :font-weight="histoConfig.style.labels.value.bold ? 'bold' : 'normal'"
                :fill="histoConfig.style.labels.value.color"
            >
                {{ histoConfig.style.labels.value.prefix }}{{ isNaN(val.value) ? '' : Number(val.value.toFixed(histoConfig.style.labels.value.rounding)).toLocaleString() }}{{ histoConfig.style.labels.value.suffix }}
            </text>

            <g v-for="(label, i) in computedDataset">
                <text 
                    :data-cy="`sparkhistogram-label-${i}`"
                    v-if="label.valueLabel"
                    :x="label.textAnchor"
                    :y="label.y + label.height + histoConfig.style.labels.valueLabel.fontSize"
                    :font-size="histoConfig.style.labels.valueLabel.fontSize"
                    text-anchor="middle"
                    :fill="histoConfig.style.labels.valueLabel.color"
                >
                    {{ label.valueLabel }}
                </text>
            </g>

            <g v-for="(time, i) in computedDataset">
                <text
                    :data-cy="`sparkhistogram-time-label-${i}`"
                    v-if="time.timeLabel"
                    :x="time.textAnchor"
                    :y="drawingArea.height - histoConfig.style.labels.timeLabel.fontSize / 2"
                    :font-size="histoConfig.style.labels.timeLabel.fontSize"
                    :fill="histoConfig.style.labels.timeLabel.color"
                    text-anchor="middle"
                >
                    {{ time.timeLabel }}
                </text>
            </g>

        </svg>
    </div>
</template>