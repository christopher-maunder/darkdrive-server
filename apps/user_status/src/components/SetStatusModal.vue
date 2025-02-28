<!--
  - @copyright Copyright (c) 2020 Georg Ehrke <oc.list@georgehrke.com>
  - @author Georg Ehrke <oc.list@georgehrke.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<NcModal size="normal"
		:title="$t('user_status', 'Set status')"
		@close="closeModal">
		<div class="set-status-modal">
			<!-- Status selector -->
			<div class="set-status-modal__header">
				<h2>{{ $t('user_status', 'Online status') }}</h2>
			</div>
			<div class="set-status-modal__online-status">
				<OnlineStatusSelect v-for="status in statuses"
					:key="status.type"
					v-bind="status"
					:checked="status.type === statusType"
					@select="changeStatus" />
			</div>

			<!-- Status message -->
			<div class="set-status-modal__header">
				<h2>{{ $t('user_status', 'Status message') }}</h2>
			</div>
			<div class="set-status-modal__custom-input">
				<CustomMessageInput ref="customMessageInput"
					:icon="icon"
					:message="message"
					@change="setMessage"
					@submit="saveStatus"
					@select-icon="setIcon" />
			</div>
			<PredefinedStatusesList @select-status="selectPredefinedMessage" />
			<ClearAtSelect :clear-at="clearAt"
				@select-clear-at="setClearAt" />
			<div class="status-buttons">
				<NcButton :wide="true"
					type="tertiary"
					:text="$t('user_status', 'Clear status message')"
					:disabled="isSavingStatus"
					@click="clearStatus">
					{{ $t('user_status', 'Clear status message') }}
				</NcButton>
				<NcButton :wide="true"
					type="primary"
					:text="$t('user_status', 'Set status message')"
					:disabled="isSavingStatus"
					@click="saveStatus">
					{{ $t('user_status', 'Set status message') }}
				</NcButton>
			</div>
		</div>
	</NcModal>
</template>

<script>
import { showError } from '@nextcloud/dialogs'
import NcModal from '@nextcloud/vue/dist/Components/NcModal'
import NcButton from '@nextcloud/vue/dist/Components/NcButton'
import { getAllStatusOptions } from '../services/statusOptionsService.js'
import OnlineStatusMixin from '../mixins/OnlineStatusMixin.js'
import PredefinedStatusesList from './PredefinedStatusesList.vue'
import CustomMessageInput from './CustomMessageInput.vue'
import ClearAtSelect from './ClearAtSelect.vue'
import OnlineStatusSelect from './OnlineStatusSelect.vue'

export default {
	name: 'SetStatusModal',

	components: {
		ClearAtSelect,
		CustomMessageInput,
		NcModal,
		OnlineStatusSelect,
		PredefinedStatusesList,
		NcButton,
	},
	mixins: [OnlineStatusMixin],

	data() {
		return {
			clearAt: null,
			icon: null,
			message: '',
			messageId: '',
			isSavingStatus: false,
			statuses: getAllStatusOptions(),
		}
	},

	/**
	 * Loads the current status when a user opens dialog
	 */
	mounted() {
		this.messageId = this.$store.state.userStatus.messageId
		this.icon = this.$store.state.userStatus.icon
		this.message = this.$store.state.userStatus.message || ''

		if (this.$store.state.userStatus.clearAt !== null) {
			this.clearAt = {
				type: '_time',
				time: this.$store.state.userStatus.clearAt,
			}
		}
	},
	methods: {
		/**
		 * Closes the Set Status modal
		 */
		closeModal() {
			this.$emit('close')
		},
		/**
		 * Sets a new icon
		 *
		 * @param {string} icon The new icon
		 */
		setIcon(icon) {
			this.messageId = null
			this.icon = icon
			this.$nextTick(() => {
				this.$refs.customMessageInput.focus()
			})
		},
		/**
		 * Sets a new message
		 *
		 * @param {string} message The new message
		 */
		setMessage(message) {
			this.messageId = null
			this.message = message
		},
		/**
		 * Sets a new clearAt value
		 *
		 * @param {object} clearAt The new clearAt object
		 */
		setClearAt(clearAt) {
			this.clearAt = clearAt
		},
		/**
		 * Sets new icon/message/clearAt based on a predefined message
		 *
		 * @param {object} status The predefined status object
		 */
		selectPredefinedMessage(status) {
			this.messageId = status.id
			this.clearAt = status.clearAt
			this.icon = status.icon
			this.message = status.message
		},
		/**
		 * Saves the status and closes the
		 *
		 * @return {Promise<void>}
		 */
		async saveStatus() {
			if (this.isSavingStatus) {
				return
			}

			try {
				this.isSavingStatus = true

				if (this.messageId !== undefined && this.messageId !== null) {
					await this.$store.dispatch('setPredefinedMessage', {
						messageId: this.messageId,
						clearAt: this.clearAt,
					})
				} else {
					await this.$store.dispatch('setCustomMessage', {
						message: this.message,
						icon: this.icon,
						clearAt: this.clearAt,
					})
				}
			} catch (err) {
				showError(this.$t('user_status', 'There was an error saving the status'))
				console.debug(err)
				this.isSavingStatus = false
				return
			}

			this.isSavingStatus = false
			this.closeModal()
		},
		/**
		 *
		 * @return {Promise<void>}
		 */
		async clearStatus() {
			try {
				this.isSavingStatus = true

				await this.$store.dispatch('clearMessage')
			} catch (err) {
				showError(this.$t('user_status', 'There was an error clearing the status'))
				console.debug(err)
				this.isSavingStatus = false
				return
			}

			this.isSavingStatus = false
			this.closeModal()
		},
	},
}
</script>

<style lang="scss" scoped>

.set-status-modal {
	padding: 8px 20px 20px 20px;

	&__header {
		text-align: center;
		font-weight: bold;
		margin: 15px 0;
	}

	&__online-status {
		display: grid;
		grid-template-columns: 1fr 1fr;
	}

	&__custom-input {
		display: flex;
		width: 100%;
		margin-bottom: 10px;
	}

	.status-buttons {
		display: flex;
		padding: 3px;
		padding-left:0;
		gap: 3px;
	}
}

@media only screen and (max-width: 500px) {
	.set-status-modal__online-status {
		grid-template-columns: none !important;
	}
}

</style>
