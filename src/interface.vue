
<template>
	<div v-if="mode" :style="customCss">
		<span class="prefix">{{ prefix }}</span>
		<span class="computed-value">{{ computedValue }}</span>
		<span class="suffix">{{ suffix }}</span>
	</div>
	<v-input
		v-else
		v-bind="$attrs"
		:field="field"
		:collection="collection"
		:primary-key="primaryKey"
		:model-value="value"
		@update:model-value="$emit('input', $event)"
	/>
	<v-notice v-if="errorMsg" type="danger">{{ errorMsg }}</v-notice>
	<div>
		<!-- Button to trigger manual computation -->
		<button @click="computeAndEmitValue">Compute Value</button>
	</div>
</template>

<script lang="ts">
import { defineComponent, ref, inject, watch, toRefs } from 'vue';
import { parseExpression } from './operations';
import { useDeepValues, useCollectionRelations } from './utils';
import { useCollection } from '@directus/extensions-sdk';

export default defineComponent({
	props: {
		value: {
			type: [String, Number],
			default: null,
		},
		field: {
			type: String,
			default: null,
		},
		type: {
			type: String,
			default: null,
		},
		collection: {
			type: String,
			default: '',
		},
		primaryKey: {
			type: [String, Number],
			default: '',
		},
		template: {
			type: String,
			default: '',
		},
		mode: {
			type: String,
			default: null,
		},
		prefix: {
			type: String,
			default: null,
		},
		suffix: {
			type: String,
			default: null,
		},
		customCss: {
			type: Object,
			default: null,
		},
		debugMode: {
			type: Boolean,
			default: false,
		},
		computeIfEmpty: {
			type: Boolean,
			default: false,
		},
		initialCompute: {
			type: Boolean,
			default: false,
		},
	},
	emits: ['input'],
	setup(props, { emit }) {
		const defaultValues = useCollection(props.collection).defaults;
		const computedValue = ref<string | number | null>(props.value);
		const relations = useCollectionRelations(props.collection);
		const { collection, field, primaryKey } = toRefs(props);
		const values = useDeepValues(
			inject<ComputedRef<Record<string, any>>>('values')!,
			relations,
			collection,
			field,
			primaryKey,
			props.template
		);
		const errorMsg = ref<string | null>(null);

		// Function to compute the value manually when the button is clicked
		function computeAndEmitValue() {
			const newValue = compute();
			if (newValue !== props.value) {
				computedValue.value = newValue;
				emit('input', newValue);
			}
		}

		function compute() {
			try {
				const res = props.template.replace(/{{.*?}}/g, (match) => {
					const expression = match.slice(2, -2).trim();
					return parseExpression(expression, values.value, defaultValues.value, props.debugMode);
				});

				errorMsg.value = null;

				if (['integer', 'decimal', 'bigInteger'].includes(props.type)) {
					return parseInt(res) || 0;
				}
				if (['float'].includes(props.type)) {
					return parseFloat(res) || 0;
				}
				return res;
			} catch (err) {
				errorMsg.value = err.message ?? 'Unknown error';
				return null;
			}
		}

		return {
			computedValue,
			errorMsg,
			computeAndEmitValue, // Return the function for button click
		};
	},
});
</script>

