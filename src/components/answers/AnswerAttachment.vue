<template>
	<div class="answer-item-text">
		<span>{{item.answer.filenameOriginal}}</span>
	</div>
</template>

<script>
export default {
	props: {
		item: {
			type: Object,
			required: true
		}
	},
	setup(props) {
		return {
			...props
		};
	}
};
</script>

<style lang="scss" scoped>
</style>