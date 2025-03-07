<template>
<XColumn :menu="menu" :column="column" :is-stacked="isStacked" @parent-focus="$event => emit('parent-focus', $event)">
	<template #header>
		<i v-if="column.tl === 'home'" class="ti ti-home"></i>
		<i v-else-if="column.tl === 'local'" class="ti ti-planet"></i>
		<i v-else-if="column.tl === 'social'" class="ti ti-rocket"></i>
		<i v-else-if="column.tl === 'global'" class="ti ti-whirl"></i>
		<span style="margin-left: 8px;">{{ column.name }}</span>
	</template>

	<div v-if="disabled" :class="$style.disabled">
		<p :class="$style.disabledTitle">
			<i class="ti ti-minus-circle"></i>
			{{ $t('disabled-timeline.title') }}
		</p>
		<p :class="$style.disabledDescription">{{ $t('disabled-timeline.description') }}</p>
	</div>
	<XTimeline v-else-if="column.tl" ref="timeline" :key="column.tl" :src="column.tl" @after="() => emit('loaded')"/>
</XColumn>
</template>

<script lang="ts" setup>
import { onMounted } from 'vue';
import XColumn from './column.vue';
import { removeColumn, updateColumn, Column } from './deck-store';
import XTimeline from '@/components/MkTimeline.vue';
import * as os from '@/os';
import { $i } from '@/account';
import { i18n } from '@/i18n';

const props = defineProps<{
	column: Column;
	isStacked: boolean;
}>();

const emit = defineEmits<{
	(ev: 'loaded'): void;
	(ev: 'parent-focus', direction: 'up' | 'down' | 'left' | 'right'): void;
}>();

let disabled = $ref(false);

onMounted(() => {
	if (props.column.tl == null) {
		setType();
	} else if ($i) {
		disabled = false; // TODO
	}
});

async function setType() {
	const { canceled, result: src } = await os.select({
		title: i18n.ts.timeline,
		items: [{
			value: 'home' as const, text: i18n.ts._timelines.home,
		}, {
			value: 'local' as const, text: i18n.ts._timelines.local,
		}, {
			value: 'social' as const, text: i18n.ts._timelines.social,
		}, {
			value: 'global' as const, text: i18n.ts._timelines.global,
		}],
	});
	if (canceled) {
		if (props.column.tl == null) {
			removeColumn(props.column.id);
		}
		return;
	}
	updateColumn(props.column.id, {
		tl: src,
	});
}

const menu = [{
	icon: 'ti ti-pencil',
	text: i18n.ts.timeline,
	action: setType,
}];
</script>

<style lang="scss" module>
.disabled {
	text-align: center;
}

.disabledTitle {
	margin: 16px;
}

.disabledDescription {
	font-size: 90%;
}
</style>
