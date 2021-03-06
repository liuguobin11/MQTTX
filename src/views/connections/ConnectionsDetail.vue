<template>
  <div class="connections-detail">
    <div class="connections-topbar right-topbar">
      <div class="connections-info">
        <div class="topbar">
          <div class="connection-head">
            <h2 :class="{ offline: !client.connected }">
              {{ titleName }}
              <a
                href="javascript:;"
                :class="['collapse-btn', showClientInfo ? 'top' : 'bottom']"
                @click="handleCollapse($route.params.id)"
              >
                <i class="el-icon-d-arrow-left"></i>
              </a>
            </h2>
          </div>
          <div class="connection-tail">
            <transition name="el-fade-in">
              <el-tooltip
                v-if="!showClientInfo && client.connected"
                placement="bottom"
                :effect="theme !== 'light' ? 'light' : 'dark'"
                :open-delay="1000"
                :content="$t('connections.disconnectedBtn')"
              >
                <a class="disconnect-btn" href="javascript:;" @click="disconnect">
                  <i v-if="!disconnectLoding" class="iconfont el-icon-switch-button"></i>
                  <i v-else class="el-icon-loading"></i>
                </a>
              </el-tooltip>
            </transition>
            <template v-if="!isNewWindow">
              <el-tooltip
                placement="bottom"
                :effect="theme !== 'light' ? 'light' : 'dark'"
                :open-delay="1000"
                :content="$t('common.config')"
              >
                <a
                  :class="['edit-btn', { disabled: client.connected }]"
                  href="javascript:;"
                  @click="handleEdit($route.params.id)"
                >
                  <i class="iconfont el-icon-edit-outline"></i>
                </a>
              </el-tooltip>
              <el-dropdown class="connection-oper" trigger="click" @command="handleCommand">
                <a href="javascript:;">
                  <i class="el-icon-more"></i>
                </a>
                <el-dropdown-menu slot="dropdown">
                  <el-dropdown-item command="searchContent">
                    <i class="iconfont icon-search"></i>{{ $t('connections.searchContent') }}
                  </el-dropdown-item>
                  <el-dropdown-item command="clearHistory">
                    <i class="iconfont icon-clear"></i>{{ $t('connections.clearHistory') }}
                  </el-dropdown-item>
                  <el-dropdown-item command="exportData">
                    <i class="el-icon-printer"></i>{{ $t('connections.exportData') }}
                  </el-dropdown-item>
                  <el-dropdown-item command="importData">
                    <i class="el-icon-upload2"></i>{{ $t('connections.importData') }}
                  </el-dropdown-item>
                  <el-dropdown-item command="disconnect" :disabled="!client.connected">
                    <i class="iconfont icon-disconnect"></i>{{ $t('connections.disconnect') }}
                  </el-dropdown-item>
                  <el-dropdown-item class="delete-item" command="deleteConnect" divided>
                    <i class="iconfont icon-delete"></i>{{ $t('connections.deleteConnect') }}
                  </el-dropdown-item>
                </el-dropdown-menu>
              </el-dropdown>
            </template>
          </div>
        </div>
        <el-collapse-transition>
          <ConnectionInfo
            v-show="showClientInfo"
            class="connection-info"
            :connection="record"
            :titleName="titleName"
            :client="client"
            :btn-loading="connectLoading"
            @handleConnect="connect"
            @handleDisconnect="disconnect"
            @handleCancel="cancel"
          />
        </el-collapse-transition>
      </div>

      <transition name="el-zoom-in-top">
        <div v-show="searchVisible" class="connections-search topbar">
          <el-input
            id="searchTopic"
            v-model="searchParams.topic"
            size="small"
            :placeholder="$t('connections.inputTopic')"
            @keyup.enter.native="searchContent"
            @keyup.esc.native="handleSearchClose"
          >
          </el-input>
          <el-input
            v-model="searchParams.payload"
            size="small"
            :placeholder="$t('connections.inputMsgContent')"
            @keyup.enter.native="searchContent"
            @keyup.esc.native="handleSearchClose"
            class="content-search"
          >
          </el-input>
          <a href="javascript:;" class="search-btn" @click="searchContent">
            <i v-if="!searchLoading" class="iconfont icon-search"></i>
            <i v-else class="el-icon-loading"></i>
          </a>
          <a href="javascript:;" class="close-search" @click="handleSearchClose">
            <i class="el-icon-circle-close"></i>
          </a>
        </div>
      </transition>
    </div>

    <div
      class="connections-detail-main right-content"
      :style="{
        paddingTop: showClientInfo ? msgTop.open : msgTop.close,
        paddingBottom: `${msgBottom}px`,
        marginLeft: showSubs ? '570px' : '341px',
      }"
    >
      <div class="connections-body">
        <div class="filter-bar" :style="{ top: showClientInfo ? bodyTop.open : bodyTop.close }">
          <span class="subs-title">
            {{ this.$t('connections.subscriptions') }}
            <a class="subs-btn" href="javascript:;" @click="handleShowSubs">
              <i class="iconfont icon-zhedie"></i>
            </a>
          </span>
          <div class="message-type">
            <el-tooltip
              placement="top"
              :effect="theme !== 'light' ? 'light' : 'dark'"
              :open-delay="500"
              :content="$t('connections.receivedPayloadDecodedBy')"
            >
              <a href="javascript:;" class="icon-tip">
                <i class="el-icon-question"></i>
              </a>
            </el-tooltip>
            <el-select class="received-type-select" size="small" v-model="receivedMsgType">
              <el-option v-for="(type, index) in payloadOptions" :key="index" :value="type"> </el-option>
            </el-select>
            <el-radio-group v-model="msgType" size="mini" @change="handleMsgTypeChanged">
              <el-radio-button label="all">{{ $t('connections.all') }}</el-radio-button>
              <el-radio-button label="received">{{ $t('connections.received') }}</el-radio-button>
              <el-radio-button label="publish">{{ $t('connections.published') }}</el-radio-button>
            </el-radio-group>
          </div>
        </div>
        <SubscriptionsList
          :subsVisible.sync="showSubs"
          :connectionId="$route.params.id"
          :record="record"
          :top="showClientInfo ? bodyTop.open : bodyTop.close"
          @onClickTopic="handleTopicClick"
          @deleteTopic="getMessages"
        />
        <div v-for="message in messages" :key="message.mid">
          <MsgLeftItem
            v-if="!message.out"
            :subsList="record.subscriptions"
            v-bind="message"
            @showmenu="handleContextMenu(arguments, message)"
          />
          <MsgRightItem v-else v-bind="message" @showmenu="handleContextMenu(arguments, message)" />
        </div>
        <contextmenu :visible.sync="showContextmenu" v-bind="contextmenuConfig">
          <a href="javascript:;" class="context-menu__item" @click="handleCopyMessage">
            <i class="el-icon-document-copy"></i>{{ $t('common.copy') }}
          </a>
          <a href="javascript:;" class="context-menu__item danger" @click="handleDeleteMessage">
            <i class="iconfont icon-delete"></i>{{ $t('common.delete') }}
          </a>
        </contextmenu>
      </div>

      <div class="connections-footer" :style="{ marginLeft: showSubs ? '570px' : '341px' }">
        <ResizeHeight v-model="inputHeight" />
        <MsgPublish
          :editor-height="inputHeight - 75"
          :subs-visible="showSubs"
          :style="{ height: `${inputHeight}px` }"
          @handleSend="sendMessage"
        />
      </div>
    </div>

    <ExportData :visible.sync="showExportData" :connection="record" />
    <ImportData :visible.sync="showImportData" @updateData="$emit('reload')" />
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator'
import { Getter, Action } from 'vuex-class'
import { TranslateResult } from 'vue-i18n'
import { ipcRenderer } from 'electron'
import mqtt, { MqttClient, IClientOptions } from 'mqtt'
import { v4 as uuidv4 } from 'uuid'
import _ from 'lodash'

import {
  deleteConnection,
  deleteMessage,
  updateConnection,
  updateConnectionMessage,
  loadConnection,
  loadConnections,
} from '@/utils/api/connection'
import time from '@/utils/time'
import matchMultipleSearch from '@/utils/matchMultipleSearch'
import topicMatch, { matchTopicMethod } from '@/utils/topicMatch'
import { getClientOptions, getMQTTProtocol } from '@/utils/mqttUtils'

import MsgRightItem from '@/components/MsgRightItem.vue'
import MsgLeftItem from '@/components/MsgLeftItem.vue'
import MsgPublish from '@/components/MsgPublish.vue'
import SubscriptionsList from '@/components/SubscriptionsList.vue'
import ResizeHeight from '@/components/ResizeHeight.vue'
import ConnectionInfo from './ConnectionInfo.vue'
import Contextmenu from '@/components/Contextmenu.vue'
import ExportData from '@/components/ExportData.vue'
import ImportData from '@/components/ImportData.vue'

import { ConnectionModel, MessageModel, SSLPath, SSLContent, ContextmenuModel } from './types'

type MessageType = 'all' | 'received' | 'publish'
type CommandType = 'searchContent' | 'clearHistory' | 'disconnect' | 'deleteConnect' | 'exportData' | 'importData'
type PayloadConvertType = 'base64' | 'hex'

interface Top {
  open: string
  close: string
}

@Component({
  components: {
    MsgRightItem,
    MsgLeftItem,
    ConnectionInfo,
    MsgPublish,
    SubscriptionsList,
    ResizeHeight,
    Contextmenu,
    ExportData,
    ImportData,
  },
})
export default class ConnectionsDetail extends Vue {
  @Prop({ required: true }) public record!: ConnectionModel

  @Action('CHANGE_SUBSCRIPTIONS') private changeSubs!: (payload: Subscriptions) => void
  @Action('CHANGE_ACTIVE_CONNECTION') private changeActiveConnection!: (payload: Client) => void
  @Action('REMOVE_ACTIVE_CONNECTION') private removeActiveConnection!: (payload: ActiveConnection) => void
  @Action('SHOW_CLIENT_INFO') private changeShowClientInfo!: (payload: ClientInfo) => void
  @Action('SHOW_SUBSCRIPTIONS') private changeShowSubscriptions!: (payload: SubscriptionsVisible) => void
  @Action('UNREAD_MESSAGE_COUNT_INCREMENT') private unreadMessageIncrement!: (payload: UnreadMessage) => void

  @Getter('activeConnection') private activeConnection: $TSFixed
  @Getter('showSubscriptions') private showSubscriptions!: boolean
  @Getter('maxReconnectTimes') private maxReconnectTimes!: number
  @Getter('currentTheme') private theme!: Theme
  @Getter('showClientInfo') private clientInfoVisibles!: {
    [id: string]: boolean
  }

  private showSubs = true
  private showClientInfo = true
  private showExportData = false
  private showImportData = false
  private connectLoading = false
  private searchVisible = false
  private searchLoading = false
  private disconnectLoding = false
  private receivedMsgType: PayloadType = 'Plaintext'
  private msgType: MessageType = 'all'
  private client: Partial<MqttClient> = {
    connected: false,
  }
  private messages: MessageModel[] = this.record.messages
  private searchParams = {
    topic: '',
    payload: '',
  }
  private titleName: string = this.record.name
  private retryTimes = 0
  private inputHeight = 155
  private msgBottom = 160
  private activeTopic = ''
  private mqttVersionDict = {
    '3.1.1': 4,
    '5.0': 5,
  }
  private payloadOptions: PayloadType[] = ['Plaintext', 'Base64', 'JSON', 'Hex']
  private showContextmenu: boolean = false
  private selectedMessage: MessageModel | null = null
  private contextmenuConfig: ContextmenuModel = {
    top: 0,
    left: 0,
  }
  private selectedInfo: string = ''

  public connect(): boolean | void {
    if (this.client.connected) {
      return false
    }
    this.connectLoading = true
    this.client = this.createClient()
    const { id } = this.record
    if (id && this.client.on) {
      this.client.on('connect', this.onConnect)
      this.client.on('error', this.onError)
      this.client.on('reconnect', this.onReConnect)
      this.client.on('close', this.onClose)
      this.client.on('message', this.onMessageArrived(id))
    }
  }

  public removeConnection(currentConnection?: ConnectionModel) {
    let { id, name } = this.record
    if (currentConnection) {
      id = currentConnection.id
      name = currentConnection.name
    }
    const confirmDelete: string = this.$t('common.confirmDelete', { name }) as string
    this.$confirm(confirmDelete, this.$t('common.warning') as string, {
      type: 'warning',
    })
      .then(async () => {
        const res: ConnectionModel | null = await deleteConnection(id as string)
        if (res) {
          this.$emit('delete')
          this.$message.success(this.$t('common.deleteSuccess') as string)
          this.removeActiveConnection({ id: res.id as string })
        }
      })
      .catch((error) => {
        // ignore(error)
      })
  }

  get bodyTop(): Top {
    return {
      open: '254px',
      close: '60px',
    }
  }

  get msgTop(): Top {
    return {
      open: '286px',
      close: '88px',
    }
  }

  get connectUrl(): string {
    const { host, port, ssl, path } = this.record
    const protocol = getMQTTProtocol(this.record)
    let url = `${protocol}://${host}:${port}`
    if (protocol === 'ws' || protocol === 'wss') {
      url = `${url}${path.startsWith('/') ? '' : '/'}${path}`
    }
    return url
  }

  get isNewWindow(): boolean {
    return this.$route.name === 'newWindow'
  }

  @Watch('record')
  private handleRecordChanged() {
    const id: string = this.$route.params.id
    this.titleName = this.record.name
    this.getConnectionValue(id)
    this.getMessages()
  }

  @Watch('inputHeight')
  private handleInputHeight(val: number) {
    const oldInputHeight = 155
    const oldMsgBottom = 160
    const offset = val - oldInputHeight
    this.msgBottom = oldMsgBottom + offset
  }

  private handleContextMenu(msgItemInfo: IArguments, message: MessageModel) {
    const [payload, event] = msgItemInfo
    if (!this.showContextmenu) {
      const { clientX, clientY } = event
      const width = window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth
      const height = window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight
      this.contextmenuConfig.left = width - clientX < 95 ? clientX - 75 : clientX
      this.contextmenuConfig.top = height - clientY < 77 ? clientY - 77 : clientY
      this.showContextmenu = true
      this.selectedMessage = message
      this.selectedInfo = payload
    } else {
      this.showContextmenu = false
    }
  }

  private handleCopyMessage() {
    if (this.selectedInfo) {
      this.$copyText(this.selectedInfo).then(
        () => {
          this.$message.success(this.$t('common.copySuccess') as string)
        },
        () => {
          this.$message.error(this.$t('common.copyFailed') as string)
        },
      )
    }
  }

  private async handleDeleteMessage() {
    const connectID = this.record.id
    let mid = ''
    if (this.selectedMessage) {
      mid = this.selectedMessage.mid
    }
    const res: ConnectionModel | null = await deleteMessage(connectID as string, mid as string)
    if (res) {
      this.showContextmenu = false
      this.$message.success(this.$t('common.deleteSuccess') as string)
      await this.$emit('reload')
    } else {
      this.showContextmenu = false
      this.$message.error(this.$t('common.deletefailed') as string)
    }
  }

  private getConnectionValue(id: string) {
    const currentActiveConnection:
      | {
          id?: string
          client: MqttClient
        }
      | undefined = this.activeConnection[id]
    const $clientInfoVisible: boolean | undefined = this.clientInfoVisibles[id]
    if ($clientInfoVisible === undefined) {
      this.showClientInfo = true
    } else {
      this.showClientInfo = $clientInfoVisible
    }
    this.showSubs = this.showSubscriptions
    if (currentActiveConnection) {
      this.client = currentActiveConnection.client
      this.setClientsMessageListener()
    } else {
      this.client = {
        connected: false,
      }
    }
  }

  private handleShowSubs() {
    this.showSubs = !this.showSubs
    this.changeShowSubscriptions({ showSubscriptions: this.showSubs })
  }

  private handleCollapse(id: string) {
    this.showClientInfo = !this.showClientInfo
    this.changeShowClientInfo({
      id,
      showClientInfo: this.showClientInfo,
    })
  }

  private handleCommand(command: CommandType) {
    switch (command) {
      case 'disconnect':
        this.disconnect()
        break
      case 'deleteConnect':
        this.removeConnection()
        break
      case 'clearHistory':
        this.handleMsgClear()
        break
      case 'searchContent':
        this.handleSearchOpen()
        break
      case 'exportData':
        this.handleExportData()
        break
      case 'importData':
        this.handleImportData()
        break
      default:
        break
    }
  }

  private handleEdit(id: string): boolean | void {
    if (this.client.connected) {
      return false
    }
    this.$router.push({
      path: `/recent_connections/${id}`,
      query: { oper: 'edit' },
    })
  }

  private getMessages() {
    this.msgType = 'all'
    this.messages = _.cloneDeep(this.record.messages)
  }
  private handleMsgClear() {
    this.messages = []
    this.record.messages = []
    this.changeActiveConnection({
      id: this.$route.params.id,
      client: this.client,
      messages: this.messages,
    })
    updateConnection(this.record.id as string, this.record)
  }
  private async handleMsgTypeChanged(type: MessageType) {
    const setChangedMessages = (changedType: MessageType, msgData: MessageModel[]) => {
      if (type === 'received') {
        this.messages = msgData.filter(($: MessageModel) => !$.out)
      } else if (type === 'publish') {
        this.messages = msgData.filter(($: MessageModel) => $.out)
      } else {
        this.messages = msgData.slice()
      }
    }
    if (this.activeTopic !== '') {
      const res = await topicMatch(this.record.messages, this.activeTopic)
      if (res) {
        setChangedMessages(type, res)
      } else {
        this.messages = [].slice()
      }
    } else {
      setChangedMessages(type, this.record.messages)
    }
  }
  private async searchContent() {
    const { topic, payload } = this.searchParams
    if (!topic && !payload) {
      return
    }
    this.searchLoading = true
    setTimeout(() => {
      this.searchLoading = false
    }, 500)
    this.getMessages()
    if (topic !== '' || payload !== '') {
      const $messages =
        this.activeTopic === '' ? _.cloneDeep(this.messages) : await topicMatch(this.record.messages, this.activeTopic)
      const res = await matchMultipleSearch($messages, this.searchParams)
      if (res) {
        this.messages = res.slice()
      } else {
        this.messages = [].slice()
      }
    }
  }
  private async handleTopicClick(sub: SubscriptionModel, reset: boolean) {
    this.getMessages()
    if (reset) {
      this.activeTopic = ''
      this.searchContent()
      return false
    }
    this.activeTopic = sub.topic
    const $messages = _.cloneDeep(this.messages)
    const res = await topicMatch($messages, sub.topic)
    if (res) {
      this.messages = res.slice()
    } else {
      this.messages = [].slice()
    }
    this.searchContent()
  }
  private handleSearchOpen() {
    this.searchVisible = true
    const $el = document.getElementById('searchTopic')
    this.$nextTick(() => {
      if ($el) {
        $el.focus()
      }
    })
  }
  private async handleSearchClose() {
    this.searchVisible = false
    this.searchParams = {
      topic: '',
      payload: '',
    }
    this.getMessages()
    if (this.activeTopic) {
      const $messages = _.cloneDeep(this.messages)
      const res = await topicMatch($messages, this.activeTopic)
      if (res) {
        this.messages = res.slice()
      } else {
        this.messages = [].slice()
      }
    }
    this.scrollToBottom()
  }

  private createClient(): MqttClient {
    const options: IClientOptions = getClientOptions(this.record)
    return mqtt.connect(this.connectUrl, options)
  }
  private cancel() {
    this.connectLoading = false
    this.client.end!(true)
    this.retryTimes = 0
  }
  private disconnect(): boolean | void {
    if (!this.client.connected) {
      return false
    }
    this.disconnectLoding = true
    const { id } = this.$route.params
    if (this.record.clean) {
      this.record.subscriptions = []
      this.changeSubs({ id, subscriptions: [] })
      updateConnection(id, this.record)
    }
    this.client.end!(false, () => {
      this.disconnectLoding = false
      this.retryTimes = 0
      this.changeActiveConnection({
        id,
        client: this.client,
        messages: this.record.messages,
      })
      this.$notify({
        title: this.$t('connections.disconnected') as string,
        message: '',
        type: 'success',
        duration: 3000,
        offset: 30,
      })
      if (!this.showClientInfo) {
        this.setShowClientInfo(true)
      }
      this.$emit('reload')
    })
  }
  private onConnect() {
    this.connectLoading = false
    this.changeActiveConnection({
      id: this.$route.params.id,
      client: this.client,
      messages: this.record.messages,
    })
    this.$notify({
      title: this.$t('connections.connected') as string,
      message: '',
      type: 'success',
      duration: 3000,
      offset: 30,
    })
    this.setShowClientInfo(false)
    this.$emit('reload')
  }
  private onError(error: string) {
    let msgTitle = this.$t('connections.connectFailed') as string
    if (error) {
      msgTitle = error
    }
    this.client.end!(true)
    this.retryTimes = 0
    this.connectLoading = false
    this.$notify({
      title: msgTitle,
      message: '',
      type: 'error',
      duration: 3000,
      offset: 30,
    })
    this.$emit('reload')
  }
  private onReConnect() {
    if (!this.record.reconnect) {
      this.client.end!(true)
      this.retryTimes = 0
      this.connectLoading = false
      this.$notify({
        title: this.$t('connections.connectFailed') as string,
        message: '',
        type: 'error',
        duration: 3000,
        offset: 30,
      })
      this.$emit('reload')
    } else {
      if (this.retryTimes > this.maxReconnectTimes) {
        this.client.end!(true)
        this.retryTimes = 0
        this.connectLoading = false
      } else {
        this.retryTimes += 1
        this.connectLoading = true
        this.$notify({
          title: this.$t('connections.reconnect') as string,
          message: '',
          type: 'warning',
          duration: 3000,
          offset: 30,
        })
      }
    }
  }
  private onClose() {
    this.connectLoading = false
  }
  private async isSearchMessage(oneMessage: MessageModel): Promise<boolean> {
    const res = await matchMultipleSearch([oneMessage], this.searchParams)
    return res && res.length ? true : false
  }
  private scrollToBottom = () => {
    setTimeout(() => {
      window.scrollTo(0, document.body.scrollHeight + 160)
    }, 100)
  }
  private onMessageArrived(id: string) {
    return (topic: string, payload: Buffer, packet: SubscriptionModel) => {
      const $payload = this.convertPayloadByType(payload, this.receivedMsgType, 'receive') as string
      const receivedMessage: MessageModel = {
        mid: uuidv4(),
        out: false,
        createAt: time.getNowDate(),
        topic,
        payload: $payload,
        qos: packet.qos,
        retain: packet.retain as boolean,
      }
      const connectionId = this.$route.params.id
      let _id = id
      if (id === connectionId) {
        this.record.messages.push({ ...receivedMessage })
        _id = connectionId
        const isActiveTopicMessages = matchTopicMethod(this.activeTopic, topic)
        const { topic: searchTopic, payload: searchPayload } = this.searchParams
        if (searchTopic || searchPayload) {
          this.isSearchMessage(receivedMessage).then((res) => {
            if (res) {
              this.messages.push(receivedMessage)
              this.scrollToBottom()
            }
          })
          return
        }
        if (this.msgType !== 'publish' && !this.activeTopic) {
          this.messages.push(receivedMessage)
        } else if (this.activeTopic && isActiveTopicMessages && this.msgType !== 'publish') {
          this.messages.push(receivedMessage)
        }
      } else {
        this.unreadMessageIncrement({ id })
      }
      ipcRenderer.send('saveMessages', {
        id: _id,
        receivedMessage,
      })
      this.scrollToBottom()
    }
  }
  private sendMessage(message: MessageModel, type: PayloadType): void | boolean {
    if (!this.client.connected) {
      this.$notify({
        title: this.$t('connections.notConnect') as string,
        message: '',
        type: 'error',
        duration: 3000,
        offset: 30,
      })
      return false
    }
    const { mid, topic, qos, payload, retain } = message
    if (!topic) {
      this.$message.warning(this.$t('connections.topicReuired') as string)
      return false
    }
    const $payload = this.convertPayloadByType(payload, type, 'publish')
    this.client.publish!(topic, $payload, { qos, retain }, (error: Error) => {
      if (error) {
        const errorMsg = error.toString()
        this.$message.error(errorMsg)
        return false
      }
      const publishMessage: MessageModel = {
        mid,
        out: true,
        createAt: time.getNowDate(),
        topic,
        payload,
        qos,
        retain,
      }
      const isActiveTopicMessages = matchTopicMethod(this.activeTopic, topic)
      const { topic: searchTopic, payload: searchPayload } = this.searchParams
      this.record.messages.push({ ...publishMessage })
      updateConnectionMessage(this.record.id as string, { ...publishMessage })
      if (searchTopic || searchPayload) {
        this.isSearchMessage(publishMessage).then((res) => {
          if (res) {
            this.messages.push(publishMessage)
            this.scrollToBottom()
          }
        })
        return
      }
      if (this.msgType !== 'received' && !this.activeTopic) {
        this.messages.push(publishMessage)
      } else if (this.activeTopic && isActiveTopicMessages && this.msgType !== 'received') {
        this.messages.push(publishMessage)
      }
      this.scrollToBottom()
    })
  }

  private setShowClientInfo(show: boolean) {
    setTimeout(() => {
      this.showClientInfo = show
      this.changeShowClientInfo({
        id: this.record.id as string,
        showClientInfo: this.showClientInfo,
      })
    }, 500)
  }

  private convertPayloadByType(value: Buffer | string, type: PayloadType, way: 'publish' | 'receive'): Buffer | string {
    const validJSONType = (jsonValue: string, warnMessage: TranslateResult) => {
      try {
        JSON.parse(jsonValue)
      } catch (error) {
        this.$message.warning(`${warnMessage} ${error.toString()}`)
      }
    }
    const genPublishPayload = (publishType: PayloadType, publishValue: string) => {
      if (publishType === 'Base64' || publishType === 'Hex') {
        const $type = publishType.toLowerCase() as PayloadConvertType
        return Buffer.from(publishValue, $type)
      }
      if (publishType === 'JSON') {
        validJSONType(publishValue, this.$t('connections.publishMsg'))
      }
      return publishValue
    }
    const genReceivePayload = (receiveType: PayloadType, receiveValue: Buffer) => {
      if (receiveType === 'Base64' || receiveType === 'Hex') {
        const $type = receiveType.toLowerCase() as 'base64' | 'hex'
        return receiveValue.toString($type)
      }
      if (receiveType === 'JSON') {
        validJSONType(receiveValue.toString(), this.$t('connections.receivedMsg'))
      }
      return receiveValue.toString()
    }
    if (way === 'publish' && typeof value === 'string') {
      return genPublishPayload(type, value)
    } else if (way === 'receive' && typeof value !== 'string') {
      return genReceivePayload(type, value)
    }
    return value
  }

  private handleExportData() {
    this.showExportData = true
  }

  private handleImportData() {
    this.showImportData = true
  }

  private created() {
    const { id } = this.$route.params
    this.getConnectionValue(id)
    ipcRenderer.on('searchContent', () => {
      this.handleSearchOpen()
    })
  }

  private setClientsMessageListener() {
    // Register connected clients message event listeners
    Object.keys(this.activeConnection).forEach((connectionID: string) => {
      const $connection: {
        id?: string
        client: MqttClient
      } = this.activeConnection[connectionID]
      const client: MqttClient = $connection.client
      let msgEventCount = 0
      if (client.listenerCount) {
        msgEventCount = client.listenerCount('message')
      }
      if (client.connected && client.on && msgEventCount === 0) {
        client.on('message', this.onMessageArrived(connectionID))
      }
      this.changeActiveConnection({
        id: connectionID,
        client,
        messages: this.messages,
      })
    })
  }

  private removeClinetsMessageListener() {
    // Remove connected clients message event listeners
    Object.keys(this.activeConnection).forEach((connectionID: string) => {
      const currentActiveConnection: {
        id?: string
        client: MqttClient
      } = this.activeConnection[connectionID]
      const client: MqttClient = currentActiveConnection.client
      if (client.removeAllListeners) {
        client.removeAllListeners('message')
      }
    })
  }

  private beforeDestroy() {
    ipcRenderer.removeAllListeners('searchContent')
    this.removeClinetsMessageListener()
  }
}
</script>

<style lang="scss" scope>
@import '~@/assets/scss/variable.scss';
@import '~@/assets/scss/mixins.scss';

.connections-detail {
  .connections-topbar {
    border-bottom: 1px solid var(--color-border-default);
    .connections-info {
      background-color: var(--color-bg-normal);
      .topbar {
        border-bottom: 0px;
        -webkit-app-region: drag;
      }
      .connection-head {
        .offline {
          color: var(--color-text-light);
        }
        a.collapse-btn {
          font-size: 18px;
          float: right;
          margin-left: 12px;
          margin-top: -1px;
        }
        @include collapse-btn-transform(90deg, -90deg);
      }
      .connection-tail {
        i {
          font-size: 18px;
        }
        .disconnect-btn {
          margin-right: 10px;
          color: var(--color-second-red);
        }
        .edit-btn {
          &.disabled {
            cursor: not-allowed;
            color: var(--color-text-light);
          }
          margin-right: 10px;
        }
        .el-dropdown.connection-oper {
          a {
            width: 24px;
            display: inline-block;
            text-align: center;
            position: relative;
            top: -1px;
          }
        }
      }
      .connection-info {
        padding: 0 16px;
      }
    }
    .connections-search {
      padding: 13px 16px 13px 16px;
      height: auto;
      background-color: var(--color-bg-normal);
      &.topbar {
        border-bottom: 0px;
        min-height: 0px;
      }
      .el-input {
        .el-input__inner {
          background: var(--color-bg-primary);
        }
      }
      .content-search {
        margin: 0 19px 0 20px;
      }
      .search-btn {
        color: var(--color-text-default);
        margin-right: 10px;
        .icon-search,
        .el-icon-loading {
          font-size: 18px;
          line-height: 32px;
        }
      }
      .el-icon-circle-close {
        font-size: 16px;
        color: var(--color-text-default);
      }
    }
  }

  .connections-detail-main {
    height: 100%;
    transition: all 0.5s;
    .connections-body {
      padding: 16px;
      .filter-bar {
        padding: 12px 16px;
        background: var(--color-bg-primary);
        position: fixed;
        left: 341px;
        right: 0;
        z-index: 1;
        transition: all 0.4s;
        .subs-title {
          color: var(--color-text-title);
          position: absolute;
          top: 15px;
        }
        .subs-btn {
          position: relative;
          top: 1px;
          left: 3px;
          .icon-zhedie {
            display: inline-block;
            transform: rotate(180deg);
          }
        }
        .message-type {
          @include flex-space-between;
          .received-type-select {
            width: 110px;
            margin-left: 245px;
          }
          .icon-tip {
            position: absolute;
            left: 239px;
            font-size: 16px;
            color: var(--color-text-tips);
          }
        }
      }
    }
    .connections-footer {
      transition: all 0.4s ease;
      position: fixed;
      width: inherit;
      bottom: 0;
      left: 0;
      right: 0;
    }
  }
}
.el-popper {
  li.delete-item {
    color: var(--color-second-red);
    &:hover {
      color: var(--color-second-red);
      background: var(--color-third-red);
    }
  }
  .el-icon-printer,
  .el-icon-upload {
    font-size: 16px;
  }
}
</style>
