<!--
 - @copyright Copyright (c) 2019 Joas Schilling <coding@schilljs.com>
 -
 - @author Joas Schilling <coding@schilljs.com>
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
 - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 - GNU Affero General Public License for more details.
 -
 - You should have received a copy of the GNU Affero General Public License
 - along with this program. If not, see <http://www.gnu.org/licenses/>.
 -
 -->

<template>
	<div id="user_retention" class="section">
		<h2>{{ t('user_retention', 'Account retention') }}</h2>

		<p class="settings-hint">
			{{ t('user_retention', 'Accounts are deleted when they did not log in within the given number of days. This will also delete all files and other data associated with the account.') }}
		</p>

		<p v-if="ldapBackendEnabled" class="settings-hint">
			{{ t('user_retention', 'Accounts from LDAP are deleted locally only, unless the LDAP write support app is enabled. When still available on LDAP, accounts will reappear.') }}
		</p>

		<div>
			<label>
				<CheckboxRadioSwitch :checked.sync="keepUsersWithoutLogin"
					id="keep_users_without_login"
					@update:checked="saveKeepUsersWithoutLogin">
					{{ t('user_retention', 'Keep accounts that never logged in') }}
				</CheckboxRadioSwitch>
			</label>
		</div>

		<div>
			<label>
				<span>{{ t('user_retention', 'Account disabling:') }}</span>
				<input id="user_days_disable"
					v-model="userDaysDisable"
					type="number"
					placeholder="180"
					@change="saveUserDaysDisable"> {{ t('user_retention', 'days') }}
				<em>{{ t('user_retention', '(0 to disable)') }}</em>
			</label>
		</div>

		<div>
			<label>
				<span>{{ t('user_retention', 'Account expiration:') }}</span>
				<input id="user_days"
					v-model="userDays"
					type="number"
					placeholder="180"
					@change="saveUserDays"> {{ t('user_retention', 'days') }}
				<em>{{ t('user_retention', '(0 to disable)') }}</em>
			</label>
		</div>

		<div v-if="guestsAppInstalled">
			<label>
				<span>{{ t('user_retention', 'Guest account disabling:') }}</span>
				<input id="guest_days_disable"
					v-model="guestDaysDisable"
					type="number"
					placeholder="180"
					@change="saveGuestDaysDisable"> {{ t('user_retention', 'days') }}
				<em>{{ t('user_retention', '(0 to disable)') }}</em>
			</label>
		</div>

		<div v-if="guestsAppInstalled">
			<label>
				<span>{{ t('user_retention', 'Guest account expiration:') }}</span>
				<input id="guest_days"
					v-model="guestDays"
					type="number"
					placeholder="180"
					@change="saveGuestDays"> {{ t('user_retention', 'days') }}
				<em>{{ t('user_retention', '(0 to disable)') }}</em>
			</label>
		</div>

		<div>
			<label>
				<span>{{ t('user_retention', 'Exclude groups:') }}</span>
				<Multiselect v-model="excludedGroups"
					name="exclude_group_select"
					class="exclude-groups-select"
					:options="groups"
					:placeholder="t('user_retention', 'Ignore members of these groups from retention')"
					:disabled="loading"
					:multiple="true"
					:searchable="true"
					:tag-width="60"
					:loading="loadingGroups"
					:show-no-options="false"
					:close-on-select="false"
					track-by="id"
					label="displayname"
					@search-change="searchGroup"
					@input="saveExcludedGroups" />
			</label>
		</div>
	</div>
</template>

<script>
import axios from '@nextcloud/axios'
import debounce from 'debounce'
import CheckboxRadioSwitch from '@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js'
import Multiselect from '@nextcloud/vue/dist/Components/NcMultiselect.js'
import { generateOcsUrl } from '@nextcloud/router'
import { loadState } from '@nextcloud/initial-state'
import {
	showError,
	showSuccess,
} from '@nextcloud/dialogs'
import '@nextcloud/dialogs/dist/index.css'

export default {
	name: 'AdminSettings',

	components: {
		CheckboxRadioSwitch,
		Multiselect,
	},

	data() {
		return {
			loading: false,
			loadingGroups: false,
			guestsAppInstalled: false,
			ldapBackendEnabled: false,
			groups: [],
			excludedGroups: [],
			keepUsersWithoutLogin: true,
			userDaysDisable: 0,
			userDays: 0,
			guestDaysDisable: 0,
			guestDays: 0,
		}
	},

	mounted() {
		this.loading = true

		this.keepUsersWithoutLogin = loadState('user_retention', 'keep_users_without_login')
		this.userDaysDisable = loadState('user_retention', 'user_days_disable')
		this.userDays = loadState('user_retention', 'user_days')
		this.guestDaysDisable = loadState('user_retention', 'guest_days_disable')
		this.guestDays = loadState('user_retention', 'guest_days')
		this.excludedGroups = loadState('user_retention', 'excluded_groups').sort(function(a, b) {
			return a.displayname.localeCompare(b.displayname)
		})
		this.guestsAppInstalled = loadState('user_retention', 'guests_app_installed')
		this.ldapBackendEnabled = loadState('user_retention', 'ldap_backend_enabled')
		this.groups = this.excludedGroups
		this.loading = false

		this.searchGroup('')
	},

	methods: {
		searchGroup: debounce(async function(query) {
			this.loadingGroups = true
			try {
				const response = await axios.get(generateOcsUrl('cloud/groups/details'), {
					search: query,
					limit: 20,
					offset: 0,
				})
				this.groups = response.data.ocs.data.groups.sort(function(a, b) {
					return a.displayname.localeCompare(b.displayname)
				})
			} catch (err) {
				showError(t('user_retention', 'Could not fetch groups'))
				console.error('Could not fetch groups', err)
			} finally {
				this.loadingGroups = false
			}
		}, 500),

		saveKeepUsersWithoutLogin() {
			OCP.AppConfig.setValue('user_retention', 'keep_users_without_login', this.keepUsersWithoutLogin ? 'yes' : 'no', {
				success: () => {
					showSuccess(t('user_retention', 'Setting saved'))
				},
				error: () => {
					showError(t('user_retention', 'Could not save the setting'))
				},
			})
		},

		saveUserDaysDisable() {
			OCP.AppConfig.setValue('user_retention', 'user_days_disable', this.userDaysDisable, {
				success: () => {
					showSuccess(t('user_retention', 'Setting saved'))
				},
				error: () => {
					showError(t('user_retention', 'Could not save the setting'))
				},
			})
		},

		saveUserDays() {
			OCP.AppConfig.setValue('user_retention', 'user_days', this.userDays, {
				success: () => {
					showSuccess(t('user_retention', 'Setting saved'))
				},
				error: () => {
					showError(t('user_retention', 'Could not save the setting'))
				},
			})
		},

		saveGuestDaysDisable() {
			OCP.AppConfig.setValue('user_retention', 'guest_days_disable', this.guestDaysDisable, {
				success: () => {
					showSuccess(t('user_retention', 'Setting saved'))
				},
				error: () => {
					showError(t('user_retention', 'Could not save the setting'))
				},
			})
		},

		saveGuestDays() {
			OCP.AppConfig.setValue('user_retention', 'guest_days', this.guestDays, {
				success: () => {
					showSuccess(t('user_retention', 'Setting saved'))
				},
				error: () => {
					showError(t('user_retention', 'Could not save the setting'))
				},
			})
		},

		saveExcludedGroups() {
			this.loading = true
			this.loadingGroups = true

			const groups = this.excludedGroups.map(group => {
				return group.id
			})

			OCP.AppConfig.setValue('user_retention', 'excluded_groups', JSON.stringify(groups), {
				success: () => {
					this.loading = false
					this.loadingGroups = false
					showSuccess(t('user_retention', 'Setting saved'))
				},
				error: () => {
					showError(t('user_retention', 'Could not save the setting'))
				},
			})
		},
	},
}
</script>

<style lang="scss" scoped>
	div > label {
		position: relative;
	}

	label span {
		display: inline-block;
		min-width: 175px;
		padding: 8px 0;
		vertical-align: top;
	}

	.excluded-groups-settings-content {
		display: flex;
		align-items: center;

		.excluded-groups-select {
			width: 300px;
		}
		button {
			margin-left: 10px;
		}
	}
</style>
