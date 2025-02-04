<script lang="ts">
	import { Button } from '../common'

	import { SUPPORTED_LANGUAGES, editScript, generateScript } from './lib'
	import type { SupportedLanguage } from '$lib/common'
	import { sendUserToast } from '$lib/toast'
	import type Editor from '../Editor.svelte'
	import { faCheck, faClose, faMagicWandSparkles } from '@fortawesome/free-solid-svg-icons'
	import Popup from '../common/popup/Popup.svelte'
	import { fade } from 'svelte/transition'
	import { Icon } from 'svelte-awesome'
	import { existsOpenaiResourcePath } from '$lib/stores'
	import type DiffEditor from '../DiffEditor.svelte'
	import { scriptLangToEditorLang } from '$lib/scripts'
	import type { Selection } from 'monaco-editor/esm/vs/editor/editor.api'
	import type SimpleEditor from '../SimpleEditor.svelte'

	// props
	export let iconOnly: boolean = false
	export let lang: SupportedLanguage | 'frontend'
	export let editor: Editor | SimpleEditor | undefined
	export let diffEditor: DiffEditor | undefined
	export let inlineScript = false

	// state
	let funcDesc: string = ''
	let genLoading: boolean = false
	let openaiAvailable: boolean | undefined = undefined
	let button: HTMLButtonElement | undefined
	let input: HTMLInputElement | undefined
	let generatedCode = ''
	let selection: Selection | undefined
	let isEdit = false

	async function onGenerate() {
		if (funcDesc.length <= 0) {
			return
		}
		try {
			// close popup ^^
			const elem = document.activeElement as HTMLElement
			if (elem.blur) {
				elem.blur()
			}

			genLoading = true
			if (isEdit && selection) {
				const selectedCode = editor?.getSelectedLines() || ''
				const originalCode = editor?.getCode() || ''
				const selectionGenCode = await editScript({
					language: lang,
					description: funcDesc,
					selectedCode
				})
				generatedCode = originalCode.replace(selectedCode, selectionGenCode + '\n')
			} else {
				generatedCode = await generateScript({
					language: lang,
					description: funcDesc
				})
			}
			funcDesc = ''
		} catch (err) {
			sendUserToast('Failed to generate code', true)
			console.error(err)
		} finally {
			genLoading = false
		}
	}

	function acceptDiff() {
		editor?.setCode(diffEditor?.getModified() || '')
		editor?.format()
		generatedCode = ''
	}

	function rejectDiff() {
		generatedCode = ''
	}

	function checkIfOpenaiAvailable(
		lang: SupportedLanguage | 'frontend',
		existsOpenaiResourcePath: boolean
	) {
		openaiAvailable = existsOpenaiResourcePath && SUPPORTED_LANGUAGES.has(lang)
	}

	function showDiff() {
		diffEditor?.setDiff(
			editor?.getCode() || '',
			generatedCode,
			lang === 'frontend' ? 'javascript' : scriptLangToEditorLang(lang)
		)
		diffEditor?.show()
		editor?.hide()
	}

	function hideDiff() {
		editor?.show()
		diffEditor?.hide()
	}

	function setSelectionHandler() {
		editor?.onDidChangeCursorSelection((e) => {
			selection = e.selection
		})
	}

	$: checkIfOpenaiAvailable(lang, $existsOpenaiResourcePath)

	$: input?.focus()

	$: lang && (generatedCode = '')

	$: generatedCode && showDiff()
	$: !generatedCode && hideDiff()
	$: editor && setSelectionHandler()
	$: selection && (isEdit = !selection.isEmpty())
</script>

{#if openaiAvailable}
	{#if generatedCode}
		{#if inlineScript}
			<div class="flex gap-1">
				<Button
					title="Discard generated code"
					btnClasses="!font-medium px-2 w-7"
					size="xs"
					color="red"
					on:click={rejectDiff}
					variant="contained"
				>
					<Icon data={faClose} />
				</Button><Button
					title="Accept generated code"
					btnClasses="!font-medium px-2 w-7"
					size="xs"
					color="green"
					on:click={acceptDiff}
				>
					<Icon data={faCheck} /></Button
				>
			</div>
		{:else}
			<div class="flex gap-1 px-2">
				<Button
					title="Discard generated code"
					btnClasses="!font-medium px-2"
					size="xs"
					color="red"
					on:click={rejectDiff}
					variant="contained"
					startIcon={{ icon: faClose }}
					{iconOnly}
				>
					Discard
				</Button><Button
					title="Accept generated code"
					btnClasses="!font-medium px-2"
					size="xs"
					color="green"
					on:click={acceptDiff}
					startIcon={{ icon: faCheck }}
					{iconOnly}
				>
					Accept
				</Button>
			</div>
		{/if}
	{:else if inlineScript}
		<Button
			size="lg"
			bind:element={button}
			color="light"
			btnClasses="!px-2 !bg-gray-100 hover:!bg-gray-200"
			loading={genLoading}
		>
			<Icon scale={0.8} data={faMagicWandSparkles} />
		</Button>
	{:else}
		<Button
			title="Generate code from prompt"
			btnClasses="!font-medium text-gray-600"
			size="xs"
			color="light"
			spacingSize="md"
			bind:element={button}
			startIcon={{ icon: faMagicWandSparkles }}
			{iconOnly}
			loading={genLoading}
		>
			{isEdit ? 'AI Edit' : 'AI Gen'}
		</Button>
	{/if}
	{#if !generatedCode && !genLoading}
		<Popup
			ref={button}
			options={{ placement: 'top-start' }}
			transition={fade}
			closeOn={[]}
			wrapperClasses="!z-[1002]"
			outerClasses="rounded shadow-xl bg-white border p-3 w-96"
		>
			<label class="block text-gray-900">
				<div class="pb-1 text-sm text-gray-600">Prompt</div>
				<div class="flex w-full">
					<input
						type="text"
						bind:this={input}
						bind:value={funcDesc}
						class="!w-auto grow"
						on:keypress={({ key }) => {
							if (key === 'Enter' && funcDesc.length > 0) {
								onGenerate()
							}
						}}
						placeholder={isEdit
							? 'Describe the changes you want'
							: 'Describe what the script should do'}
					/>
					<Button
						size="xs"
						color="blue"
						buttonType="button"
						btnClasses="!p-1 !w-[34px] !ml-1"
						aria-label="Generate"
						on:click={onGenerate}
						disabled={funcDesc.length <= 0}
					>
						<Icon data={faMagicWandSparkles} />
					</Button>
				</div>
			</label>
		</Popup>
	{/if}
{/if}
