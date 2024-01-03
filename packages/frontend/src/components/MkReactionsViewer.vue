<!--
SPDX-FileCopyrightText: syuilo and other misskey contributors
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<TransitionGroup
	:enterActiveClass="defaultStore.state.animation ? $style.transition_x_enterActive : ''"
	:leaveActiveClass="defaultStore.state.animation ? $style.transition_x_leaveActive : ''"
	:enterFromClass="defaultStore.state.animation ? $style.transition_x_enterFrom : ''"
	:leaveToClass="defaultStore.state.animation ? $style.transition_x_leaveTo : ''"
	:moveClass="defaultStore.state.animation ? $style.transition_x_move : ''"
	tag="div" :class="$style.root"
>
	<XReaction v-for="[reaction, count] in reactions" :key="reaction" :reaction="reaction" :count="count" :isInitial="initialReactions.has(reaction)" :note="note" @reactionToggled="onMockToggleReaction"/>
	<slot v-if="hasMoreReactions" name="more"/>
</TransitionGroup>
</template>

<script lang="ts" setup>
import * as Misskey from 'misskey-js';
import { inject, watch, ref } from 'vue';
import XReaction from '@/components/MkReactionsViewer.reaction.vue';
import { defaultStore } from '@/store.js';
import * as os from '@/os.js';
import { customEmojis, customEmojisMap } from '@/custom-emojis.js';

const props = withDefaults(defineProps<{
	note: Misskey.entities.Note;
	maxNumber?: number;
}>(), {
	maxNumber: Infinity,
});

const mock = inject<boolean>('mock', false);

const emit = defineEmits<{
	(ev: 'mockUpdateMyReaction', emoji: string, delta: number): void;
}>();

const initialReactions = new Set(Object.keys(props.note.reactions));

const reactions = ref<[string, number][]>([]);
const hasMoreReactions = ref(false);

if (props.note.myReaction && !Object.keys(reactions.value).includes(props.note.myReaction)) {
	reactions.value[props.note.myReaction] = props.note.reactions[props.note.myReaction];
}

function onMockToggleReaction(emoji: string, count: number) {
	if (!mock) return;

	const i = reactions.value.findIndex((item) => item[0] === emoji);
	if (i < 0) return;

	emit('mockUpdateMyReaction', emoji, (count - reactions.value[i][1]));
}

watch([() => props.note.reactions, () => props.maxNumber], ([newSource, maxNumber]) => {
	let newReactions: [string, number][] = [];
	hasMoreReactions.value = Object.keys(newSource).length > maxNumber;

	for (let i = 0; i < reactions.value.length; i++) {
		const reaction = reactions.value[i][0];
		if (reaction in newSource && newSource[reaction] !== 0) {
			reactions.value[i][1] = newSource[reaction];
			newReactions.push(reactions.value[i]);
		}
	}

	const newReactionsNames = newReactions.map(([x]) => x);
	newReactions = [
		...newReactions,
		...Object.entries(newSource)
			.sort(([, a], [, b]) => b - a)
			.filter(([y], i) => i < maxNumber && !newReactionsNames.includes(y)),
	];

	newReactions = newReactions.slice(0, props.maxNumber);

	if (props.note.myReaction && !newReactions.map(([x]) => x).includes(props.note.myReaction)) {
		newReactions.push([props.note.myReaction, newSource[props.note.myReaction]]);
	}

	reactions.value = newReactions;

	let emojisToConvert = [];
	let emojisToConvertLocal = [];
	for(let i = 0; i < reactions.value.length; i++){
		let name: string = reactions.value[i][0];
		if(name.startsWith(':')){
			let strippedName = name.replace(/^:(\w+)(@.*)?:$/, '$1');
			if(name.endsWith('@.:')){
				emojisToConvertLocal.push(name.substring(1, name.length - 3));
			}
			else if(!emojisToConvertLocal.includes(strippedName) && !emojisToConvert.includes(strippedName)){
				emojisToConvert.push(strippedName);
			}
		}
	}
	emojisToConvertLocal.forEach(name => {
		let count = 0;
		reactions.value.filter(e => e[0].includes(`:${name}@`)).forEach(e => count += e[1]);
		reactions.value = reactions.value.filter(e => !e[0].includes(`:${name}@`) || e[0].endsWith('@.:'));
		reactions.value.find(e => e[0] === `:${name}@.:`)[1] = count;
	});
	emojisToConvert.forEach(name => {
		const exactMatch = customEmojis.value.find(emoji => emoji.name === name);
		if (exactMatch) {
			let count = 0;
			reactions.value.filter(e => e[0].includes(`:${name}@`)).forEach(e => count += e[1]);
			reactions.value = reactions.value.filter(e => !e[0].includes(`:${name}@`) || e[0].endsWith('@.:'));
			reactions.value.find(e => e[0] === `:${name}@.:`)[1] = count;
		}

	});

}, { immediate: true, deep: true });
</script>

<style lang="scss" module>
.transition_x_move,
.transition_x_enterActive,
.transition_x_leaveActive {
	transition: opacity 0.2s cubic-bezier(0,.5,.5,1), transform 0.2s cubic-bezier(0,.5,.5,1) !important;
}
.transition_x_enterFrom,
.transition_x_leaveTo {
	opacity: 0;
	transform: scale(0.7);
}
.transition_x_leaveActive {
	position: absolute;
}

.root {
	margin: 4px -2px 0 -2px;
	cursor: auto; /* not clickToOpen-able */

	&:empty {
		display: none;
	}
}
</style>
