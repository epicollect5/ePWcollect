<template>
	<ion-card
		class="question-card"
		:class="{'animate__animated animate__fadeIn' : !isGroupInput}"
	>
		<ion-card-header class="question-label force-no-padding">
			<ion-card-title>
				<question-label-action
					:key="state.answer.answer"
					:disabled="!isFileAvailable"
					action="media"
					:questionText="state.question"
					:answer="state.answer.answer"
					@on-label-button-click="openPopover"
				></question-label-action>
			</ion-card-title>
		</ion-card-header>
		<ion-card-content
			class="ion-text-center"
			:class="{'ion-margin' : isGroupInput}"
		>
			<dropzone
				:filename="state.answer.answer"
				:filestate="state.pwaFileState"
				:fileError="state.fileError"
				:key="state.answer.answer"
				v-if="isPWA"
				:type="state.inputDetails.type"
				:inputRef="state.inputDetails.ref"
				:uuid="entryUuid"
				@file-loaded="onFileLoadedPWA"
				@file-dropped="onFileDroppedPWA"
				@file-error="onFileErrorPWA"
			></dropzone>

			<grid-question-narrow v-if="!isPWA">
				<template #content>
					<ion-button
						class="question-action-button"
						color="secondary"
						expand="block"
						@click="pick()"
					>
						<ion-icon
							slot="start"
							:icon="documentOutline"
						></ion-icon>
						{{labels.pick}}
					</ion-button>

				</template>
			</grid-question-narrow>
			<ion-item>
				<ion-label class="ion-text-center">{{state.answer.answer.filenameOriginal}}</ion-label>
			</ion-item>
			<div
				class="question-error"
				v-if="hasError"
			>
				{{errorMessage}}
			</div>
		</ion-card-content>
	</ion-card>
</template>

<script>
import { onMounted } from 'vue';
import { STRINGS } from '@/config/strings.js';
import { PARAMETERS } from '@/config';
import { useRootStore } from '@/stores/root-store';
import { documentOutline } from 'ionicons/icons';
import { reactive, computed } from '@vue/reactivity';
import { inject } from 'vue';
import { Capacitor } from '@capacitor/core';
import { popoverMediaHandler } from '@/use/questions/popover-media-handler';
import GridQuestionNarrow from '@/components/GridQuestionNarrow';
import QuestionLabelAction from '@/components/QuestionLabelAction';
import Dropzone from '@/components/Dropzone';
import { questionCommonService } from '@/services/entry/question-common-service';
import { FilePicker } from '@capawesome/capacitor-file-picker';
import { moveFileService } from '@/services/filesystem/move-file-service';
import { projectModel } from '@/models/project-model';
import { notificationService } from '@/services/notification-service';
import { utilsService } from '@/services/utilities/utils-service';

export default {
	components: {
		GridQuestionNarrow,
		QuestionLabelAction,
		Dropzone
	},
	props: {
		inputRef: {
			type: String,
			required: true
		},
		type: {
			type: String,
			required: true
		},
		isGroupInput: {
			type: Boolean,
			required: true
		}
	},
	emits: ['question-mounted'],
	setup(props, context) {
		const rootStore = useRootStore();
		const language = rootStore.language;
		const labels = STRINGS[language].labels;
		const questionType = props.type.toUpperCase();
		const entriesAddState = inject('entriesAddState');
		const entriesAddScope = rootStore.entriesAddScope;
		const state = reactive({
			inputDetails: {},
			currentInputRef: null,
			error: {
				errors: []
			},
			required: false,
			question: '',
			pattern: null,
			answer: {
				answer: {
					filenameRenamed: '',
					filenameOriginal: ''
				},
				was_jumped: false
			},
			confirmAnswer: {
				verify: false,
				answer: ''
			},
			fileSource: '',
			pwaFileState: PARAMETERS.PWA_FILE_STATE.CACHED,
			fileError: labels.unknown_error
		});

		//set up question
		questionCommonService.setUpInputParams(state, props.inputRef, entriesAddState);

		onMounted(async () => {
			console.log('Component Question is mounted, type ->', questionType);
			//emit event to entriesAddState
			context.emit('question-mounted');
		});

		const tempDir = rootStore.tempDir;
		const persistentDir = rootStore.persistentDir;
		let filenameRenamed = '';
		const source = '';

		const projectRef = entriesAddScope.entryService.entry.projectRef;
		const media = entriesAddScope.entryService.entry.media;
		// Check whether we want to index the media object using the main entry uuid, or branch entry uuid
		const entryUuid = !entriesAddState.isBranch
			? entriesAddScope.entryService.entry.entryUuid
			: entriesAddState.branchEntryService.entry.entryUuid;

		media[entryUuid] = media[entryUuid] || {};

		//get saved media details if any
		if (!Object.prototype.hasOwnProperty.call(media[entryUuid], state.inputDetails.ref)) {
			media[entryUuid][state.inputDetails.ref] = {};
			media[entryUuid][state.inputDetails.ref].cached = '';
			media[entryUuid][state.inputDetails.ref].stored = '';
			media[entryUuid][state.inputDetails.ref].type = state.inputDetails.type;

			if (rootStore.isPWA) {
				media[entryUuid][state.inputDetails.ref].filenamePWA = {};
				media[entryUuid][state.inputDetails.ref].filenamePWA.cached = '';
				media[entryUuid][state.inputDetails.ref].filenamePWA.stored = '';
			}
		} else {
			if (rootStore.isPWA) {
				//load preview in dropzone
				//show cached or stored image if any, Cached image will win over stored one
				if (media[entryUuid][state.inputDetails.ref].filenamePWA.cached !== '') {
					filenameRenamed = media[entryUuid][state.inputDetails.ref].filenamePWA.cached;
					state.pwaFileState = PARAMETERS.PWA_FILE_STATE.CACHED;
				} else {
					if (media[entryUuid][state.inputDetails.ref].filenamePWA.stored !== '') {
						filenameRenamed = media[entryUuid][state.inputDetails.ref].filenamePWA.stored;
						state.pwaFileState = PARAMETERS.PWA_FILE_STATE.STORED;
					}
				}
			} else {
				//show cached or stored image if any, Cached image will win over stored one
				if (media[entryUuid][state.inputDetails.ref].cached !== '') {
					filenameRenamed = media[entryUuid][state.inputDetails.ref].cached;
				} else {
					if (media[entryUuid][state.inputDetails.ref].stored !== '') {
						filenameRenamed = media[entryUuid][state.inputDetails.ref].stored;
					}
				}
			}
		}

		state.answer.answer.filenameRenamed = filenameRenamed;

		const methods = {
			async openPopover(e) {
				const mediaFile = media[entryUuid][state.inputDetails.ref];
				if (rootStore.isPWA) {
					if (mediaFile.filenamePWA.cached === '' && mediaFile.filenamePWA.stored === '') {
						return false;
					}
				} else {
					if (mediaFile.cached === '' && mediaFile.stored === '') {
						return false;
					}
				}
				popoverMediaHandler({
					media,
					entryUuid,
					state,
					e,
					mediaType: PARAMETERS.QUESTION_TYPES.VIDEO
				});
			},
			async pick() {
				//if (rootStore.device.platform !== PARAMETERS.WEB) {
				//open file picker, filter by formats?
				const result = await FilePicker.pickFiles({
					//types: ['text/plain'],
					multiple: false
				});

				const filenameOriginal = result.files[0].name;
				const filepath = result.files[0].path;

				console.log(result);

				if (rootStore.device.platform !== PARAMETERS.WEB) {
					await notificationService.showProgressDialog(STRINGS[language].labels.wait);
					try {
						//if we do not have done any recording yet, generate a new file name
						if (media[entryUuid][state.inputDetails.ref].cached === '') {
							//check if we have a stored filename, i.e user is replacing the photo for the entry
							if (media[entryUuid][state.inputDetails.ref].stored === '') {
								//generate new file name, this is a brand new file
								filenameRenamed = utilsService.generateMediaFilename(
									entryUuid,
									PARAMETERS.QUESTION_TYPES.ATTACHMENT,
									filenameOriginal.split('.')[1] //extension
								);
							} else {
								//use stored filename
								filenameRenamed = media[entryUuid][state.inputDetails.ref].stored;
							}
							media[entryUuid][state.inputDetails.ref].cached = filenameRenamed;
						} else {
							//use the cached path not to fill the cache with a new file all the time
							filenameRenamed = media[entryUuid][state.inputDetails.ref].cached;
						}

						state.answer.answer.filenameRenamed = filenameRenamed;

						await moveFileService.copyToAppTemporaryDir(
							filepath,
							state.answer.answer.filenameRenamed,
							PARAMETERS.QUESTION_TYPES.ATTACHMENT,
							projectModel.getProjectRef()
						);
					} catch (error) {
						console.log(error);
						//todo: remove references from media object?
					} finally {
						state.answer.answer.filenameOriginal = filenameOriginal;
						state.answer.answer.filenameRenamed = filenameRenamed;
						notificationService.hideProgressDialog();
					}
				}
			},
			//file uploaded to server or stored file loaded
			onFileLoadedPWA(filename) {
				//todo:
				state.answer.answer = filename;
			},
			onFileDroppedPWA(filename) {
				//todo:
				//if a file is dropped, we have a cached file to show
				//(cached takes priority on stored, if any)
				media[entryUuid][state.inputDetails.ref].filenamePWA.cached = filename;
				state.pwaFileState = PARAMETERS.PWA_FILE_STATE.CACHED;
				state.answer.answer = filename;
			},
			onFileErrorPWA(error) {
				state.fileError = error;
			}
		};

		const computedScope = {
			isFileAvailable: computed(() => {
				const mediaFile = media[entryUuid][state.inputDetails.ref];

				if (rootStore.isPWA) {
					return mediaFile.filenamePWA.cached !== '' || mediaFile.filenamePWA.stored !== '';
				}
				return mediaFile.cached !== '' || mediaFile.stored !== '';
			}),
			isPWA: computed(() => {
				return rootStore.isPWA;
			}),
			hasError: computed(() => {
				//any error for this question?
				return utilsService.questionHasError(state);
			}),
			errorMessage: computed(() => {
				if (Object.keys(state.error.errors).length > 0) {
					return state.error?.errors[state.currentInputRef]?.message;
				} else {
					return '';
				}
			})
		};

		return {
			labels,
			state,
			entryUuid,
			...methods,
			...props,
			...computedScope,
			//icons
			documentOutline
		};
	}
};
</script>

<style lang="scss" scoped>
</style>