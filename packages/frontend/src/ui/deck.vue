<template>
<div :class="[$style.root, { [$style.rootIsMobile]: isMobile }]">
	<XSidebar v-if="!isMobile"/>

	<div :class="$style.main">
		<XStatusBars/>
		<div ref="columnsEl" :class="[$style.columns, deckStore.reactiveState.columnAlign.value]" @contextmenu.self.prevent="onContextmenu">
			<template v-for="ids in layout">
				<!-- sectionを利用しているのは、deck.vue側でcolumnに対してfirst-of-typeを効かせるため -->
				<section
					v-if="ids.length > 1"
					:class="$style.folder"
					:style="columns.filter(c => ids.includes(c.id)).some(c => c.flexible) ? { flex: 1, minWidth: '350px' } : { width: Math.max(...columns.filter(c => ids.includes(c.id)).map(c => c.width)) + 'px' }"
				>
					<DeckColumnCore v-for="id in ids" :ref="id" :key="id" :column="columns.find(c => c.id === id)" :is-stacked="true" @parent-focus="moveFocus(id, $event)"/>
				</section>
				<DeckColumnCore
					v-else
					:ref="ids[0]"
					:key="ids[0]"
					:class="$style.column"
					:column="columns.find(c => c.id === ids[0])"
					:is-stacked="false"
					:style="columns.find(c => c.id === ids[0])!.flexible ? { flex: 1, minWidth: '350px' } : { width: columns.find(c => c.id === ids[0])!.width + 'px' }"
					@parent-focus="moveFocus(ids[0], $event)"
				/>
			</template>
			<div v-if="layout.length === 0" class="_panel" :class="$style.onboarding">
				<div>{{ i18n.ts._deck.introduction }}</div>
				<MkButton primary style="margin: 1em auto;" @click="addColumn">{{ i18n.ts._deck.addColumn }}</MkButton>
				<div>{{ i18n.ts._deck.introduction2 }}</div>
			</div>
			<div :class="$style.sideMenu">
				<div :class="$style.sideMenuTop">
					<button v-tooltip.noDelay.left="`${i18n.ts._deck.profile}: ${deckStore.state.profile}`" :class="$style.sideMenuButton" class="_button" @click="changeProfile"><i class="ti ti-caret-down"></i></button>
					<button v-tooltip.noDelay.left="i18n.ts._deck.deleteProfile" :class="$style.sideMenuButton" class="_button" @click="deleteProfile"><i class="ti ti-trash"></i></button>
				</div>
				<div :class="$style.sideMenuMiddle">
					<button v-tooltip.noDelay.left="i18n.ts._deck.addColumn" :class="$style.sideMenuButton" class="_button" @click="addColumn"><i class="ti ti-plus"></i></button>
				</div>
				<div :class="$style.sideMenuBottom">
					<button v-tooltip.noDelay.left="i18n.ts.settings" :class="$style.sideMenuButton" class="_button" @click="showSettings"><i class="ti ti-settings"></i></button>
				</div>
			</div>
		</div>
	</div>

	<div v-if="isMobile" :class="$style.nav">
		<button :class="$style.navButton" class="_button" @click="drawerMenuShowing = true"><i :class="$style.navButtonIcon" class="ti ti-menu-2"></i><span v-if="menuIndicated" :class="$style.navButtonIndicator"><i class="_indicatorCircle"></i></span></button>
		<button :class="$style.navButton" class="_button" @click="mainRouter.push('/')"><i :class="$style.navButtonIcon" class="ti ti-home"></i></button>
		<button :class="$style.navButton" class="_button" @click="mainRouter.push('/my/notifications')"><i :class="$style.navButtonIcon" class="ti ti-bell"></i><span v-if="$i?.hasUnreadNotification" :class="$style.navButtonIndicator"><i class="_indicatorCircle"></i></span></button>
		<button :class="$style.postButton" class="_button" @click="os.post()"><i :class="$style.navButtonIcon" class="ti ti-pencil"></i></button>
	</div>

	<Transition
		:enter-active-class="$store.state.animation ? $style.transition_menuDrawerBg_enterActive : ''"
		:leave-active-class="$store.state.animation ? $style.transition_menuDrawerBg_leaveActive : ''"
		:enter-from-class="$store.state.animation ? $style.transition_menuDrawerBg_enterFrom : ''"
		:leave-to-class="$store.state.animation ? $style.transition_menuDrawerBg_leaveTo : ''"
	>
		<div
			v-if="drawerMenuShowing"
			:class="$style.menuBg"
			class="_modalBg"
			@click="drawerMenuShowing = false"
			@touchstart.passive="drawerMenuShowing = false"
		></div>
	</Transition>

	<Transition
		:enter-active-class="$store.state.animation ? $style.transition_menuDrawer_enterActive : ''"
		:leave-active-class="$store.state.animation ? $style.transition_menuDrawer_leaveActive : ''"
		:enter-from-class="$store.state.animation ? $style.transition_menuDrawer_enterFrom : ''"
		:leave-to-class="$store.state.animation ? $style.transition_menuDrawer_leaveTo : ''"
	>
		<div v-if="drawerMenuShowing" :class="$style.menu">
			<XDrawerMenu/>
		</div>
	</Transition>

	<XCommon/>
</div>
</template>

<script lang="ts" setup>
import { computed, defineAsyncComponent, ref, watch } from 'vue';
import { v4 as uuid } from 'uuid';
import XCommon from './_common_/common.vue';
import { deckStore, addColumn as addColumnToStore, loadDeck, getProfiles, deleteProfile as deleteProfile_ } from './deck/deck-store';
import DeckColumnCore from '@/ui/deck/column-core.vue';
import XSidebar from '@/ui/_common_/navbar.vue';
import XDrawerMenu from '@/ui/_common_/navbar-for-mobile.vue';
import MkButton from '@/components/MkButton.vue';
import { getScrollContainer } from '@/scripts/scroll';
import * as os from '@/os';
import { navbarItemDef } from '@/navbar';
import { $i } from '@/account';
import { i18n } from '@/i18n';
import { mainRouter } from '@/router';
import { unisonReload } from '@/scripts/unison-reload';
const XStatusBars = defineAsyncComponent(() => import('@/ui/_common_/statusbars.vue'));

mainRouter.navHook = (path, flag): boolean => {
	if (flag === 'forcePage') return false;
	const noMainColumn = !deckStore.state.columns.some(x => x.type === 'main');
	if (deckStore.state.navWindow || noMainColumn) {
		os.pageWindow(path);
		return true;
	}
	return false;
};

const isMobile = ref(window.innerWidth <= 500);
window.addEventListener('resize', () => {
	isMobile.value = window.innerWidth <= 500;
});

const drawerMenuShowing = ref(false);

const route = 'TODO';
watch(route, () => {
	drawerMenuShowing.value = false;
});

const columns = deckStore.reactiveState.columns;
const layout = deckStore.reactiveState.layout;
const menuIndicated = computed(() => {
	if ($i == null) return false;
	for (const def in navbarItemDef) {
		if (navbarItemDef[def].indicated) return true;
	}
	return false;
});

function showSettings() {
	os.pageWindow('/settings/deck');
}

let columnsEl = $shallowRef<HTMLElement>();

const addColumn = async (ev) => {
	const columns = [
		'main',
		'widgets',
		'notifications',
		'tl',
		'antenna',
		'list',
		'channel',
		'mentions',
		'direct',
	];

	const { canceled, result: column } = await os.select({
		title: i18n.ts._deck.addColumn,
		items: columns.map(column => ({
			value: column, text: i18n.t('_deck._columns.' + column),
		})),
	});
	if (canceled) return;

	addColumnToStore({
		type: column,
		id: uuid(),
		name: i18n.t('_deck._columns.' + column),
		width: 330,
	});
};

const onContextmenu = (ev) => {
	os.contextMenu([{
		text: i18n.ts._deck.addColumn,
		action: addColumn,
	}], ev);
};

document.documentElement.style.overflowY = 'hidden';
document.documentElement.style.scrollBehavior = 'auto';
window.addEventListener('wheel', (ev) => {
	if (ev.target === columnsEl && ev.deltaX === 0) {
		columnsEl.scrollLeft += ev.deltaY;
	} else if (getScrollContainer(ev.target as HTMLElement) == null && ev.deltaX === 0) {
		columnsEl.scrollLeft += ev.deltaY;
	}
});
loadDeck();

function moveFocus(id: string, direction: 'up' | 'down' | 'left' | 'right') {
	// TODO??
}

function changeProfile(ev: MouseEvent) {
	const items = ref([{
		text: deckStore.state.profile,
		active: true.valueOf,
	}]);
	getProfiles().then(profiles => {
		items.value = [{
			text: deckStore.state.profile,
			active: true.valueOf,
		}, ...(profiles.filter(k => k !== deckStore.state.profile).map(k => ({
			text: k,
			action: () => {
				deckStore.set('profile', k);
				unisonReload();
			},
		}))), null, {
			text: i18n.ts._deck.newProfile,
			icon: 'ti ti-plus',
			action: async () => {
				const { canceled, result: name } = await os.inputText({
					title: i18n.ts._deck.profile,
					allowEmpty: false,
				});
				if (canceled) return;

				deckStore.set('profile', name);
				unisonReload();
			},
		}];
	});
	os.popupMenu(items, ev.currentTarget ?? ev.target);
}

async function deleteProfile() {
	const { canceled } = await os.confirm({
		type: 'warning',
		text: i18n.t('deleteAreYouSure', { x: deckStore.state.profile }),
	});
	if (canceled) return;

	deleteProfile_(deckStore.state.profile);
	deckStore.set('profile', 'default');
	unisonReload();
}
</script>

<style lang="scss" module>
.transition_menuDrawerBg_enterActive,
.transition_menuDrawerBg_leaveActive {
	opacity: 1;
	transition: opacity 300ms cubic-bezier(0.23, 1, 0.32, 1);
}
.transition_menuDrawerBg_enterFrom,
.transition_menuDrawerBg_leaveTo {
	opacity: 0;
}

.transition_menuDrawer_enterActive,
.transition_menuDrawer_leaveActive {
	opacity: 1;
	transform: translateX(0);
	transition: transform 300ms cubic-bezier(0.23, 1, 0.32, 1), opacity 300ms cubic-bezier(0.23, 1, 0.32, 1);
}
.transition_menuDrawer_enterFrom,
.transition_menuDrawer_leaveTo {
	opacity: 0;
	transform: translateX(-240px);
}

.root {
	$nav-hide-threshold: 650px; // TODO: どこかに集約したい

	--margin: var(--marginHalf);

	--deckDividerThickness: 5px;

	display: flex;
	height: 100dvh;
	box-sizing: border-box;
	flex: 1;
}

.rootIsMobile {
	padding-bottom: 100px;
}

.main {
	flex: 1;
	min-width: 0;
	display: flex;
	flex-direction: column;
}

.columns {
	flex: 1;
	display: flex;
	overflow-x: auto;
	overflow-y: clip;

	&.center {
		> .column:first-of-type {
			margin-left: auto;
		}

		> .column:last-of-type {
			margin-right: auto;
		}
	}
}

.column {
	flex-shrink: 0;
	border-right: solid var(--deckDividerThickness) var(--deckDivider);

	&:first-of-type {
		border-left: solid var(--deckDividerThickness) var(--deckDivider);
	}
}

.folder {
	composes: column;
	display: flex;
	flex-direction: column;

	> *:not(:last-of-type) {
		border-bottom: solid var(--deckDividerThickness) var(--deckDivider);
	}
}

.onboarding {
	padding: 32px;
	height: min-content;
	text-align: center;
	margin: auto;
}

.sideMenu {
	flex-shrink: 0;
	margin-right: 0;
	margin-left: auto;
	display: flex;
	flex-direction: column;
	justify-content: center;
	width: 32px;
}

.sideMenuButton {
	display: block;
	width: 100%;
	aspect-ratio: 1;
}

.sideMenuTop {
	margin-bottom: auto;
}

.sideMenuMiddle {
	margin-top: auto;
	margin-bottom: auto;
}

.sideMenuBottom {
	margin-top: auto;
}

.menuBg {
	z-index: 1001;
}

.menu {
	position: fixed;
	top: 0;
	left: 0;
	z-index: 1001;
	height: 100dvh;
	width: 240px;
	box-sizing: border-box;
	contain: strict;
	overflow: auto;
	overscroll-behavior: contain;
	background: var(--navBg);
}

.nav {
	position: fixed;
	z-index: 1000;
	bottom: 0;
	left: 0;
	padding: 12px 12px max(12px, env(safe-area-inset-bottom, 0px)) 12px;
	display: grid;
	grid-template-columns: 1fr 1fr 1fr 1fr;
	grid-gap: 8px;
	width: 100%;
	box-sizing: border-box;
	-webkit-backdrop-filter: var(--blur, blur(32px));
	backdrop-filter: var(--blur, blur(32px));
	background-color: var(--header);
	border-top: solid 0.5px var(--divider);
}

.navButton {
	position: relative;
	padding: 0;
	aspect-ratio: 1;
	width: 100%;
	max-width: 60px;
	margin: auto;
	border-radius: 100%;
	background: var(--panel);
	color: var(--fg);

	&:hover {
		background: var(--panelHighlight);
	}

	&:active {
		background: var(--X2);
	}
}

.postButton {
	composes: navButton;
	background: linear-gradient(90deg, var(--buttonGradateA), var(--buttonGradateB));
	color: var(--fgOnAccent);

	&:hover {
		background: linear-gradient(90deg, var(--X8), var(--X8));
	}

	&:active {
		background: linear-gradient(90deg, var(--X8), var(--X8));
	}
}

.navButtonIcon {
	font-size: 18px;
	vertical-align: middle;
}

.navButtonIndicator {
	position: absolute;
	top: 0;
	left: 0;
	color: var(--indicator);
	font-size: 16px;
	animation: blink 1s infinite;
}
</style>
