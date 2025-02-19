<template>
	<AppContentDetails>
		<Loading v-if="loading" />
		<Error
			v-else-if="error"
			:error="error && error.message ? error.message : t('mail', 'Not found')"
			:message="errorMessage"
			:data="error" />
		<Composer
			v-else
			:from-account="composerData.accountId"
			:to="composerData.to"
			:cc="composerData.cc"
			:bcc="composerData.bcc"
			:subject="composerData.subject"
			:body="composerData.body"
			:draft="saveDraft"
			:send="sendMessage"
			:reply-to="composerData.replyTo"
			:forward-from="composerData.forwardFrom" />
	</AppContentDetails>
</template>

<script>
import AppContentDetails from '@nextcloud/vue/dist/Components/AppContentDetails'
import Axios from '@nextcloud/axios'
import { generateUrl } from '@nextcloud/router'
import { showWarning } from '@nextcloud/dialogs'
import { translate as t } from '@nextcloud/l10n'

import { buildForwardSubject, buildReplySubject, buildRecipients as buildReplyRecipients } from '../ReplyBuilder'
import Composer from './Composer'
import Error from './Error'
import { getRandomMessageErrorMessage } from '../util/ErrorMessageFactory'
import { detect, html, plain, toPlain } from '../util/text'
import Loading from './Loading'
import logger from '../logger'
import { saveDraft, sendMessage } from '../service/MessageService'

export default {
	name: 'NewMessageDetail',
	components: {
		AppContentDetails,
		Composer,
		Error,
		Loading,
	},
	data() {
		return {
			loading: false,
			draft: undefined,
			original: undefined,
			originalBody: undefined,
			errorMessage: '',
			error: undefined,
			newDraftId: undefined,
		}
	},
	computed: {
		composerData() {
			if (this.draft !== undefined) {
				logger.info('todo: handle draft data', { draft: this.draft })
				return {
					to: this.draft.to,
					cc: this.draft.cc,
					bcc: this.draft.bcc,
					subject: this.draft.subject,
					body: this.draft.hasHtmlBody ? html(this.draft.body) : plain(this.draft.body),
				}
			} else if (this.$route.query.messageId !== undefined) {
				// Forward or reply to a message
				const message = this.original
				logger.debug('forwarding or replying to message', { message })

				if (this.$route.params.threadId === 'reply') {
					logger.debug('simple reply', {
						message,
					})

					return {
						accountId: message.accountId,
						to: message.from,
						cc: [],
						subject: buildReplySubject(message.subject),
						body: this.originalBody,
						originalBody: this.originalBody,
						replyTo: message,
					}
				} else if (this.$route.params.threadId === 'replyAll') {
					logger.debug('replying to all', { original: this.original })
					const account = this.$store.getters.getAccount(message.accountId)
					const recipients = buildReplyRecipients(message, {
						email: account.emailAddress,
						label: account.name,
					})

					return {
						accountId: message.accountId,
						to: recipients.to,
						cc: recipients.cc,
						subject: buildReplySubject(message.subject),
						body: this.originalBody,
						originalBody: this.originalBody,
						replyTo: message,
					}
				} else if (this.$route.params.threadId === 'asNew') {
					logger.debug('composing as new', { original: this.original })

					if (this.original.attachments.length) {
						showWarning(t('mail', 'Attachments were not copied. Please add them manually.'))
					}

					return {
						accountId: message.accountId,
						to: message.to,
						cc: message.cc,
						subject: message.subject,
						body: this.originalBody,
						originalBody: this.originalBody,
					}
				} else {
					// forward
					return {
						accountId: message.accountId,
						to: [],
						cc: [],
						subject: buildForwardSubject(message.subject),
						body: this.originalBody,
						originalBody: this.originalBody,
						forwardFrom: message,
					}
				}
			} else {
				// New or mailto: message
				logger.debug('composing a new message or handling a mailto link', {
					threadId: this.$route.params.threadId,
				})

				let accountId
				// Only preselect an account when we're not in a unified mailbox
				if (this.$route.params.accountId !== 0 && this.$route.params.accountId !== '0') {
					accountId = parseInt(this.$route.params.accountId, 10)
				}

				return {
					accountId,
					to: this.stringToRecipients(this.$route.query.to),
					cc: this.stringToRecipients(this.$route.query.cc),
					subject: this.$route.query.subject || '',
					body: this.$route.query.body ? detect(this.$route.query.body) : html(''),
				}
			}
		},
	},
	watch: {
		$route(to, from) {
			// `saveDraft` replaced the current URL with the updated draft UID
			// in that case we don't really start a new draft but just keep the
			// URL consistent, hence not loading anything
			if (to.name === 'message' && this.draft
				&& (
					to.params.draftId === parseInt(this.draft.databaseId, 10)
					|| to.params.draftId === this.newDraftId
				)
			) {
				logger.debug('detected navigation to current (new) draft UID, not reloading')
				return
			}
			logger.debug('the  draft ID changed, we have to fetch the draft', {
				currentId: this?.draft?.databaseId ?? 'no draft currently loaded',
				newId: to.params.draftId,
			})

			this.fetchMessage()
		},
	},
	created() {
		this.fetchMessage()
	},
	methods: {
		stringToRecipients(str) {
			if (str === undefined) {
				return []
			}

			return [
				{
					label: str,
					email: str,
				},
			]
		},
		fetchMessage() {
			if (this.$route.params.draftId !== undefined) {
				return this.fetchDraftMessage(this.$route.params.draftId)
			} else if (this.$route.query.messageId !== undefined) {
				return this.fetchOriginalMessage(this.$route.query.messageId)
			}
		},
		fetchDraftMessage(id) {
			this.loading = true
			this.draft = undefined
			this.error = undefined
			this.errorMessage = ''

			this.$store
				.dispatch('fetchMessage', id)
				.then((draft) => {
					if (draft.databaseId !== parseInt(this.$route.params.draftId, 10)) {
						logger.debug("User navigated away, loaded draft won't be shown", {
							draft,
							draftId: this.$route.params.draftId,
						})
						return
					}

					this.draft = draft

					if (this.draft === undefined) {
						logger.info('draft could not be found', { id })
						this.errorMessage = getRandomMessageErrorMessage()
						this.loading = false
						return
					}

					this.loading = false
				})
				.catch((error) => {
					logger.error(`could not load draft ${id}`, { error })
					if (error.isError) {
						this.errorMessage = t('mail', 'Could not load your draft')
						this.error = error
						this.loading = false
					}
				})
		},
		async fetchOriginalMessage(id) {
			this.loading = true
			this.error = undefined
			this.errorMessage = ''

			logger.debug(`fetching original message ${id}`)

			try {
				const message = await this.$store.dispatch('fetchMessage', id)
				if (message.databaseId !== parseInt(this.$route.query.messageId, 10)) {
					logger.debug("User navigated away, loaded original message won't be used", {
						message,
						messageId: message.databaseId,
						urlId: this.$route.query.messageId,
					})
					return
				}

				logger.debug('original message fetched', { message })
				this.original = message

				let body = plain(message.body || '')
				if (message.hasHtmlBody) {
					logger.debug('original message has HTML body')
					const resp = await Axios.get(
						generateUrl('/apps/mail/api/messages/{id}/html?plain=true', {
							id,
						})
					)

					body = html(resp.data)
				}
				this.originalBody = body
			} catch (error) {
				logger.error('could not load original message ' + id, { error })
				if (error.isError) {
					this.errorMessage = t('mail', 'Could not load original message')
					this.error = error
					this.loading = false
				}
			} finally {
				this.loading = false
			}
		},
		async saveDraft(data) {
			if (data.draftId === undefined && this.draft) {
				logger.debug('draft data does not have a draftId, adding one', { draft: this.draft, data, id: this.draft.databaseId })
				data.draftId = this.draft.databaseId
			}
			const dataForServer = {
				...data,
				body: data.isHtml ? data.body.value : toPlain(data.body).value,
			}
			const { id } = await saveDraft(data.account, dataForServer)

			// Remove old draft envelope
			this.$store.commit('removeEnvelope', { id: data.draftId })
			this.$store.commit('removeMessage', { id: data.draftId })

			// Fetch new draft envelope
			await this.$store.dispatch('fetchEnvelope', id)

			// Update route to new draft (actual redirect will be skipped)
			const account = this.$store.getters.getAccount(data.account)
			if (parseInt(this.$route.params.mailboxId, 10) === account.draftsMailboxId) {
				this.newDraftId = id
				this.$router.replace({
					to: 'message',
					params: {
						mailboxId: this.$route.params.mailboxId,
						threadId: this.$route.params.threadId,
						draftId: id,
					},
				})
			}

			return id
		},
		async sendMessage(data) {
			logger.debug('sending message', { data })
			const dataForServer = {
				...data,
				body: data.isHtml ? data.body.value : toPlain(data.body).value,
			}
			await sendMessage(data.account, dataForServer)

			// Remove old draft envelope
			this.$store.commit('removeEnvelope', { id: data.draftId })
			this.$store.commit('removeMessage', { id: data.draftId })
		},
	},
}
</script>
