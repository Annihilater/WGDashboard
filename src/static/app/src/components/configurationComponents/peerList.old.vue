<script>
import PeerSearch from "@/components/configurationComponents/peerSearch.vue";
import {DashboardConfigurationStore} from "@/stores/DashboardConfigurationStore.js";
import {WireguardConfigurationsStore} from "@/stores/WireguardConfigurationsStore.js";
import {fetchGet} from "@/utilities/fetch.js";
import Peer from "@/components/configurationComponents/peer.vue";
import { Line, Bar } from 'vue-chartjs'
import Fuse from "fuse.js";
import {
	Chart,
	LineElement,
	BarElement, 
	BarController,
	LineController,
	LinearScale,
	Legend,
	Title,
	Tooltip,
	CategoryScale,
	PointElement
} from 'chart.js';
Chart.register(
	LineElement,
	BarElement,
	BarController,
	LineController,
	LinearScale,
	Legend,
	Title,
	Tooltip,
	CategoryScale,
	PointElement
);
import dayjs from "dayjs";
import {defineAsyncComponent, ref} from "vue";
import LocaleText from "@/components/text/localeText.vue";
import PeerRow from "@/components/configurationComponents/peerRow.vue";
import {GetLocale} from "@/utilities/locale.js";
import PeerSearchBar from "@/components/configurationComponents/peerSearchBar.vue";
import ProtocolBadge from "@/components/protocolBadge.vue";
import EditRawConfigurationFile
	from "@/components/configurationComponents/editConfigurationComponents/editRawConfigurationFile.vue";
import PeerIntersectionObserver from "@/components/configurationComponents/peerIntersectionObserver.vue";
export default {
	name: "peerList",
	components: {
		PeerIntersectionObserver,
		EditRawConfigurationFile,
		ProtocolBadge,
		PeerSearchBar,
		PeerRow,
		DeleteConfiguration:
			defineAsyncComponent(() => import("@/components/configurationComponents/deleteConfiguration.vue")),
		ConfigurationBackupRestore:
			defineAsyncComponent(() => import("@/components/configurationComponents/configurationBackupRestore.vue")),
		SelectPeers:
			defineAsyncComponent(() => import("@/components/configurationComponents/selectPeers.vue")),
		EditConfiguration:
			defineAsyncComponent(() => import("@/components/configurationComponents/editConfiguration.vue")),
		LocaleText,
		PeerShareLinkModal:
			defineAsyncComponent(() => import("@/components/configurationComponents/peerShareLinkModal.vue")),
		PeerJobsLogsModal: 
			defineAsyncComponent(() => import("@/components/configurationComponents/peerJobsLogsModal.vue")),
		PeerJobsAllModal:
			defineAsyncComponent(() => import("@/components/configurationComponents/peerJobsAllModal.vue")),
		PeerJobs: 
			defineAsyncComponent(() => import("@/components/configurationComponents/peerJobs.vue")), 
		PeerCreate:
			defineAsyncComponent(() => import("@/components/configurationComponents/peerCreate.vue")),
		PeerQRCode:
			defineAsyncComponent(() => import("@/components/configurationComponents/peerQRCode.vue")),
		PeerConfigurationFile:
			defineAsyncComponent(() => import("@/components/configurationComponents/peerConfigurationFile.vue")),
		PeerSettings:
			defineAsyncComponent(() => import("@/components/configurationComponents/peerSettings.vue")), 
		PeerSearch, 
		Peer, 
		Line, 
		Bar
	},
	async setup(){
		const dashboardConfigurationStore = DashboardConfigurationStore();
		const wireguardConfigurationStore = WireguardConfigurationsStore();
		const interval = ref(undefined)
		return {dashboardConfigurationStore, wireguardConfigurationStore, interval}
	},
	data(){
		return {
			configurationToggling: false,
			loading: false,
			error: null,
			configurationInfo: [],
			configurationPeers: [],
			historyDataSentDifference: [],
			historyDataReceivedDifference: [],
			historySentData: {
				labels: [],
				datasets: [
					{
						label: 'Data Sent',
						data: [],
						fill: false,
						borderColor: '#198754',
						tension: 0
					},
				],
			},
			historyReceiveData: {
				labels: [],
				datasets: [
					{
						label: 'Data Received',
						data: [],
						fill: false,
						borderColor: '#0d6efd',
						tension: 0
					},
				],
			},
			peerSetting: {
				modalOpen: false,
				selectedPeer: undefined
			},
			peerScheduleJobs:{
				modalOpen: false,
				selectedPeer: undefined
			},
			peerQRCode: {
				modalOpen: false,
				peerConfigData: undefined
			},
			peerConfigurationFile: {
				modalOpen: false,
				peerConfigData: undefined
			},
			peerCreate: {
				modalOpen: false
			},
			peerScheduleJobsAll: {
				modalOpen: false
			},
			peerScheduleJobsLogs: {
				modalOpen: false
			},
			peerShare:{
				modalOpen: false,
				selectedPeer: undefined
			},
			editConfiguration: {
				modalOpen: false
			},
			selectPeers: {
				modalOpen: false
			},
			backupRestore: {
				modalOpen: false
			},
			deleteConfiguration: {
				modalOpen: false
			},
			editRawConfigurationFile: {
				modalOpen: false
			},
			peerSearchBarShow: false,
			searchStringTimeout: undefined,
			searchString: "",
			showPeersCount: 10,
			showPeersThreshold: 10,
			observer: undefined
		}
	},
	watch: {
		'$route': {
			immediate: true,
			handler(){
				this.showPeersCount = 20;
				clearInterval(this.dashboardConfigurationStore.Peers.RefreshInterval);
				this.loading = true;
				let id = this.$route.params.id;
				this.configurationInfo = [];
				this.configurationPeers = [];
				if (id){
					this.getPeers(id)
					this.setPeerInterval();
				}
			}
		},
		'dashboardConfigurationStore.Configuration.Server.dashboard_refresh_interval'(){
			clearInterval(this.dashboardConfigurationStore.Peers.RefreshInterval);
			this.setPeerInterval();
		}
	},
	beforeRouteLeave(){
		clearInterval(this.dashboardConfigurationStore.Peers.RefreshInterval);
	},
	methods:{
		toggle(){
			this.configurationToggling = true;
			fetchGet("/api/toggleWireguardConfiguration", {
				configurationName: this.configurationInfo.Name
			}, (res) => {
				if (res.status){
					this.dashboardConfigurationStore.newMessage("Server",
						`${this.configurationInfo.Name} ${res.data ? 'is on':'is off'}`, "success")
				}else{
					this.dashboardConfigurationStore.newMessage("Server",
						res.message, 'danger')
				}
				this.wireguardConfigurationStore.Configurations.find(x => x.Name === this.configurationInfo.Name).Status = res.data
				console.log(this.wireguardConfigurationStore.Configurations.find(x => x.Name === this.configurationInfo.Name).Status)
				
				this.configurationInfo.Status = res.data
				this.configurationToggling = false;
			})
		},
		getPeers(id = this.$route.params.id){
			fetchGet("/api/getWireguardConfigurationInfo",
				{
					configurationName: id
				}, (res) => {
					this.configurationInfo = res.data.configurationInfo;
					this.configurationPeers = res.data.configurationPeers;
					this.configurationPeers.forEach(x => {
						x.restricted = false;
					})
					res.data.configurationRestrictedPeers.forEach(x => {
						x.restricted = true;
						this.configurationPeers.push(x)
					})
					this.loading = false;
					if (this.configurationPeers.length > 0){
						const sent = this.configurationPeers.map(x => x.total_sent + x.cumu_sent)
							.reduce((x,y) => x + y).toFixed(4);
						const receive = this.configurationPeers.map(x => x.total_receive + x.cumu_receive).reduce((x,y) => x + y).toFixed(4);
						if (
							this.historyDataSentDifference[this.historyDataSentDifference.length - 1] !== sent
						){
							if (this.historyDataSentDifference.length > 0){
								this.historySentData = {
									labels: [...this.historySentData.labels, dayjs().format("HH:mm:ss A")],
									datasets: [
										{
											label: 'Data Sent',
											data: [...this.historySentData.datasets[0].data,
												((sent - this.historyDataSentDifference[this.historyDataSentDifference.length - 1])*1000)
													.toFixed(4)],
											fill: false,
											borderColor: '#198754',
											tension: 0
										}
									],
								}
							}
							this.historyDataSentDifference.push(sent)
						}
						if (
							this.historyDataReceivedDifference[this.historyDataReceivedDifference.length - 1] !== receive
						){
							if (this.historyDataReceivedDifference.length > 0){
								this.historyReceiveData = {
									labels: [...this.historyReceiveData.labels, dayjs().format("HH:mm:ss A")],
									datasets: [
										{
											label: 'Data Received',
											data: [...this.historyReceiveData.datasets[0].data,
												((receive - this.historyDataReceivedDifference[this.historyDataReceivedDifference.length - 1])*1000)
													.toFixed(4)],
											fill: false,
											borderColor: '#0d6efd',
											tension: 0
										}
									],
								}
							}
							this.historyDataReceivedDifference.push(receive)
						}
					}
				});
			
		},
		setPeerInterval(){
			this.dashboardConfigurationStore.Peers.RefreshInterval = setInterval(() => {
				this.getPeers()
			}, parseInt(this.dashboardConfigurationStore.Configuration.Server.dashboard_refresh_interval))
		},
	},
	computed: {
		configurationSummary(){
			return {
				connectedPeers: this.configurationPeers.filter(x => x.status === "running").length,
				totalUsage: this.configurationPeers.length > 0 ? 
					this.configurationPeers.filter(x => !x.restricted)
						.map(x => x.total_data + x.cumu_data).reduce((a, b) => a + b, 0).toFixed(4) : 0,
				totalReceive: this.configurationPeers.length > 0 ? 
					this.configurationPeers.filter(x => !x.restricted)
						.map(x => x.total_receive + x.cumu_receive).reduce((a, b) => a + b, 0).toFixed(4) : 0,
				totalSent: this.configurationPeers.length > 0 ? 
					this.configurationPeers.filter(x => !x.restricted)
						.map(x => x.total_sent + x.cumu_sent).reduce((a, b) => a + b, 0).toFixed(4) : 0
			}
		},
		receiveData(){
			return this.historyReceiveData
		},
		sentData(){
			return this.historySentData
		},
		individualDataUsage(){
			return {
				labels: this.configurationPeers.map(x => {
					if (x.name) return x.name
					return `Untitled Peer - ${x.id}`
				}),
				datasets: [{
					label: 'Total Data Usage',
					data: this.configurationPeers.map(x => x.cumu_data + x.total_data),
					backgroundColor: this.configurationPeers.map(x => `#0dcaf0`),
					tooltip: {
						callbacks: {
							label: (tooltipItem) => {
								return `${tooltipItem.formattedValue} GB`
							}
						}
					}
				}]
			}
		},
		individualDataUsageChartOption(){
			return {
				responsive: true,
				plugins: {
					legend: {
						display: false
					}
				},
				scales: {
					x: {
						ticks: {
							display: false,
						},
						grid: {
							display: false
						},
					},
					y:{
						ticks: {
							callback: (val, index) => {
								return `${val} GB`
							}
						},
						grid: {
							display: false
						},
					}
				}
			}
		},
		chartOptions() {
			return {
				responsive: true,
				plugins: {
					legend: {
						display: false
					},
					tooltip: {
						callbacks: {
							label: (tooltipItem) => {
								return `${tooltipItem.formattedValue} MB/s`
							}
						}
					}
				},
				scales: {
					x: {
						ticks: {
							display: false,
						},
						grid: {
							display: false
						},
					},
					y:{
						ticks: {
							callback: (val, index) => {
								return `${val} MB/s`
							}
						},
						grid: {
							display: false
						},
					}
				}
			}
		},
		searchPeers(){
			const fuse = new Fuse(this.configurationPeers, {
				keys: ["name", "id", "allowed_ip"]
			});

			const result = this.wireguardConfigurationStore.searchString ?
				this.configurationPeers.filter(x => {
					return x.name.includes(this.wireguardConfigurationStore.searchString) ||
						x.id.includes(this.wireguardConfigurationStore.searchString) || 
						x.allowed_ip.includes(this.wireguardConfigurationStore.searchString)
				}) : this.configurationPeers;
			
			if (this.dashboardConfigurationStore.Configuration.Server.dashboard_sort === "restricted"){
				return result.sort((a, b) => {
					if ( a[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort]
						< b[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort] ){
						return 1;
					}
					if ( a[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort]
						> b[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort]){
						return -1;
					}
					return 0;
				}).slice(0, this.showPeersCount);
			}
			
			return result.sort((a, b) => {
				if ( a[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort]
					< b[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort] ){
					return -1;
				}
				if ( a[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort]
					> b[this.dashboardConfigurationStore.Configuration.Server.dashboard_sort]){
					return 1;
				}
				return 0;
			}).slice(0, this.showPeersCount)
		},
	}
}
</script>

<template>
	<div>
		<div v-if="!this.loading" class="container-md" id="peerList">
			<div class="d-flex align-items-sm-center flex-column flex-sm-row gap-3">
				<div>
					<div class="text-muted d-flex align-items-center gap-2">
						<h5 class="mb-0">
							<ProtocolBadge :protocol="this.configurationInfo.Protocol"></ProtocolBadge>
						</h5>
					</div>
					<div class="d-flex align-items-center gap-3">
						<h1 class="mb-0 display-4"><samp>{{this.configurationInfo.Name}}</samp></h1>
					</div>
				</div>
				<div class="ms-sm-auto d-flex gap-2 flex-column">
					<div class="card rounded-3 bg-transparent ">
						<div class="card-body py-2 d-flex align-items-center">
							<small class="text-muted">
								<LocaleText t="Status"></LocaleText>
							</small>
							<div class="dot ms-2" :class="{active: this.configurationInfo.Status}"></div>
							<div class="form-check form-switch mb-0 ms-auto pe-0 me-0">
								<label class="form-check-label" style="cursor: pointer" :for="'switch' + this.configurationInfo.id">
									<LocaleText t="On" v-if="this.configurationInfo.Status && !this.configurationToggling"></LocaleText>
									<LocaleText t="Off" v-else-if="!this.configurationInfo.Status && !this.configurationToggling"></LocaleText>
									<span v-if="this.configurationToggling"
									      class="spinner-border spinner-border-sm ms-2" aria-hidden="true">
								</span>
								</label>
								<input class="form-check-input"
								       style="cursor: pointer"
								       :disabled="this.configurationToggling"
								       type="checkbox" role="switch" :id="'switch' + this.configurationInfo.id"
								       @change="this.toggle()"
								       v-model="this.configurationInfo.Status"
								>
							</div>

						</div>
					</div>
					<div class="d-flex gap-2">
						<RouterLink
							to="create"
							class="titleBtn py-2 text-decoration-none btn text-primary-emphasis bg-primary-subtle rounded-3 border-1 border-primary-subtle ">
							<i class="bi bi-plus-lg me-2"></i>
							<LocaleText t="Peer"></LocaleText>
						</RouterLink>
						<button class="titleBtn py-2 text-decoration-none btn text-primary-emphasis bg-primary-subtle rounded-3 border-1 border-primary-subtle "
						        @click="editConfiguration.modalOpen = true"
						        type="button" aria-expanded="false">
							<i class="bi bi-gear-fill me-2"></i>
							<LocaleText t="Configuration Settings"></LocaleText>
						</button>
					</div>
				</div>
			</div>
			<hr>
			<div class="row mt-3 gy-2 gx-2 mb-2">
				<div class="col-12 col-lg-3">
					<div class="card rounded-3 bg-transparent  h-100">
						<div class="card-body py-2 d-flex flex-column justify-content-center">
							<p class="mb-0 text-muted"><small>
								<LocaleText t="Address"></LocaleText>
							</small></p>
							{{this.configurationInfo.Address}}
						</div>
					</div>
				</div>
				<div class="col-12 col-lg-3">
					<div class="card rounded-3 bg-transparent   h-100">
						<div class="card-body py-2 d-flex flex-column justify-content-center">
							<p class="mb-0 text-muted"><small>
								<LocaleText t="Listen Port"></LocaleText>
							</small></p>
							{{this.configurationInfo.ListenPort}}
						</div>
					</div>
				</div>

				<div style="word-break: break-all" class="col-12 col-lg-6">
					<div class="card rounded-3 bg-transparent   h-100">
						<div class="card-body py-2 d-flex flex-column justify-content-center">
							<p class="mb-0 text-muted"><small>
								<LocaleText t="Public Key"></LocaleText>
							</small></p>
							<samp>{{this.configurationInfo.PublicKey}}</samp>
						</div>
					</div>
				</div>
			</div>
			<div class="row gx-2 gy-2 mb-2">
				<div class="col-12 col-lg-3">
					<div class="card rounded-3 bg-transparent  h-100">
						<div class="card-body d-flex">
							<div>
								<p class="mb-0 text-muted"><small>
									<LocaleText t="Connected Peers"></LocaleText>
								</small></p>
								<strong class="h4">
									{{configurationSummary.connectedPeers}} / {{configurationPeers.length}}
								</strong>
							</div>
							<i class="bi bi-ethernet ms-auto h2 text-muted"></i>
						</div>
					</div>
				</div>
				<div class="col-12 col-lg-3">
					<div class="card rounded-3 bg-transparent  h-100">
						<div class="card-body d-flex">
							<div>
								<p class="mb-0 text-muted"><small>
									<LocaleText t="Total Usage"></LocaleText>
								</small></p>
								<strong class="h4">{{configurationSummary.totalUsage}} GB</strong>
							</div>
							<i class="bi bi-arrow-down-up ms-auto h2 text-muted"></i>
						</div>
					</div>
				</div>
				<div class="col-12 col-lg-3">
					<div class="card rounded-3 bg-transparent  h-100">
						<div class="card-body d-flex">
							<div>
								<p class="mb-0 text-muted"><small>
									<LocaleText t="Total Received"></LocaleText>
								</small></p>
								<strong class="h4 text-primary">{{configurationSummary.totalReceive}} GB</strong>
							</div>
							<i class="bi bi-arrow-down ms-auto h2 text-muted"></i>
						</div>
					</div>
				</div>
				<div class="col-12 col-lg-3">
					<div class="card rounded-3 bg-transparent  h-100">
						<div class="card-body d-flex">
							<div>
								<p class="mb-0 text-muted"><small>
									<LocaleText t="Total Sent"></LocaleText>
								</small></p>
								<strong class="h4 text-success">{{configurationSummary.totalSent}} GB</strong>
							</div>
							<i class="bi bi-arrow-up ms-auto h2 text-muted"></i>
						</div>
					</div>
				</div>
			</div>
			<div class="row gx-2 gy-2 mb-3">
				<div class="col-12 col-lg-6">
					<div class="card rounded-3 bg-transparent " style="height: 270px">
						<div class="card-header bg-transparent border-0">
							<small class="text-muted">
								<LocaleText t="Peers Data Usage"></LocaleText>
							</small></div>
						<div class="card-body pt-1">
							<Bar
								:data="individualDataUsage"
								:options="individualDataUsageChartOption"
								style="width: 100%; height: 200px;  max-height: 200px"></Bar>
						</div>
					</div>
				</div>
				<div class="col-sm col-lg-3">
					<div class="card rounded-3 bg-transparent " style="height: 270px">
						<div class="card-header bg-transparent border-0"><small class="text-muted">
							<LocaleText t="Real Time Received Data Usage"></LocaleText>
						</small></div>
						<div class="card-body pt-1">
							<Line
								:options="chartOptions"
								:data="receiveData"
								style="width: 100%; height: 200px; max-height: 200px"
							></Line>
						</div>
					</div>
				</div>
				<div class="col-sm col-lg-3">
					<div class="card rounded-3 bg-transparent " style="height: 270px">
						<div class="card-header bg-transparent border-0"><small class="text-muted">
							<LocaleText t="Real Time Sent Data Usage"></LocaleText>
						</small></div>
						<div class="card-body  pt-1">
							<Line
								:options="chartOptions"
								:data="sentData"
								style="width: 100%; height: 200px; max-height: 200px"
							></Line>
						</div>
					</div>
				</div>
			</div>
			<hr>
			<div style="margin-bottom: 80px">
				<PeerSearch
					@search="this.peerSearchBarShow = true"
					@jobsAll="this.peerScheduleJobsAll.modalOpen = true"
					@jobLogs="this.peerScheduleJobsLogs.modalOpen = true"
					@editConfiguration="this.editConfiguration.modalOpen = true"
					@selectPeers="this.selectPeers.modalOpen = true"
					@backupRestore="this.backupRestore.modalOpen = true"
					@deleteConfiguration="this.deleteConfiguration.modalOpen = true"
					:configuration="this.configurationInfo"></PeerSearch>
				<TransitionGroup name="peerList" tag="div" class="row gx-2 gy-2 z-0 position-relative">
					<div class="col-12 col-lg-6 col-xl-4"
					     :key="peer.id"
					     v-for="peer in this.searchPeers">
						<Peer :Peer="peer"
						      @share="this.peerShare.selectedPeer = peer.id; this.peerShare.modalOpen = true;"
						      @refresh="this.getPeers()"
						      @jobs="peerScheduleJobs.modalOpen = true; peerScheduleJobs.selectedPeer = this.configurationPeers.find(x => x.id === peer.id)"
						      @setting="peerSetting.modalOpen = true; peerSetting.selectedPeer = this.configurationPeers.find(x => x.id === peer.id)"
						      @qrcode="(file) => {this.peerQRCode.peerConfigData = file; this.peerQRCode.modalOpen = true;}"
						      @configurationFile="(file) => {this.peerConfigurationFile.peerConfigData = file; this.peerConfigurationFile.modalOpen = true;}"
						></Peer>
					</div>
				</TransitionGroup>
			</div>
			<Transition name="slideUp">
				<PeerSearchBar
					@close="peerSearchBarShow = false"
					v-if="this.peerSearchBarShow"></PeerSearchBar>
			</Transition>

			<Transition name="zoom">
				<PeerSettings v-if="this.peerSetting.modalOpen"
				              key="settings"
				              :selectedPeer="this.peerSetting.selectedPeer"
				              @refresh="this.getPeers()"
				              @close="this.peerSetting.modalOpen = false">
				</PeerSettings>
			</Transition>
			<Transition name="zoom">
				<PeerQRCode :peerConfigData="this.peerQRCode.peerConfigData"
				            key="qrcode"
				            @close="this.peerQRCode.modalOpen = false"
				            v-if="peerQRCode.modalOpen"></PeerQRCode>
			</Transition>
			<Transition name="zoom">
				<PeerJobs
					@refresh="this.getPeers()"
					v-if="this.peerScheduleJobs.modalOpen"
					:selectedPeer="this.peerScheduleJobs.selectedPeer"
					@close="this.peerScheduleJobs.modalOpen = false">
				</PeerJobs>
			</Transition>
			<Transition name="zoom">
				<PeerJobsAllModal
					v-if="this.peerScheduleJobsAll.modalOpen"
					@refresh="this.getPeers()"
					@allLogs="peerScheduleJobsLogs.modalOpen = true"
					@close="this.peerScheduleJobsAll.modalOpen = false"
					:configurationPeers="this.configurationPeers"
				>
				</PeerJobsAllModal>
			</Transition>
			<Transition name="zoom">
				<PeerJobsLogsModal v-if="this.peerScheduleJobsLogs.modalOpen"

				                   @close="this.peerScheduleJobsLogs.modalOpen = false"
				                   :configurationInfo="this.configurationInfo"
				>
				</PeerJobsLogsModal>
			</Transition>
			<Transition name="zoom">
				<PeerShareLinkModal
					v-if="this.peerShare.modalOpen"
					@close="this.peerShare.modalOpen = false; this.peerShare.selectedPeer = undefined;"
					:peer="this.configurationPeers.find(x => x.id === this.peerShare.selectedPeer)"></PeerShareLinkModal>
			</Transition>
			<Transition name="zoom">
				<EditConfiguration
					@editRaw="this.editRawConfigurationFile.modalOpen = true"
					@close="this.editConfiguration.modalOpen = false"
					@dataChanged="(d) => this.configurationInfo = d"
					@backupRestore="this.backupRestore.modalOpen = true"
					@deleteConfiguration="this.deleteConfiguration.modalOpen = true"
					:configurationInfo="this.configurationInfo"
					v-if="this.editConfiguration.modalOpen"></EditConfiguration>
			</Transition>
			<Transition name="zoom">
				<SelectPeers
					@refresh="this.getPeers()"
					v-if="this.selectPeers.modalOpen"
					:configurationPeers="this.configurationPeers"
					@close="this.selectPeers.modalOpen = false"
				></SelectPeers>
			</Transition>

			<Transition name="zoom">
				<DeleteConfiguration
					@backup="backupRestore.modalOpen = true"
					@close="deleteConfiguration.modalOpen = false"
					v-if="deleteConfiguration.modalOpen"></DeleteConfiguration>
			</Transition>
			<Transition name="zoom">
				<EditRawConfigurationFile
					@close="editRawConfigurationFile.modalOpen = false"
					v-if="editRawConfigurationFile.modalOpen">
				</EditRawConfigurationFile>
			</Transition>
			<Transition name="zoom">
				<ConfigurationBackupRestore
					@close="backupRestore.modalOpen = false"
					@refreshPeersList="this.getPeers()"
					v-if="backupRestore.modalOpen"></ConfigurationBackupRestore>
			</Transition>
			<Transition name="zoom">
				<PeerConfigurationFile
					@close="peerConfigurationFile.modalOpen = false"
					v-if="peerConfigurationFile.modalOpen"
					:configurationFile="peerConfigurationFile.peerConfigData"
				></PeerConfigurationFile>
			</Transition>
		</div>
		<PeerIntersectionObserver 
			:showPeersCount="this.showPeersCount"
			:peerListLength="this.searchPeers.length"
			@loadMore="this.showPeersCount += this.showPeersThreshold"></PeerIntersectionObserver>
	</div>
</template>

<style scoped>
.peerNav .nav-link{
	&.active{
		background-color: #efefef;
	}
}

th, td{
	background-color: transparent !important;
}

@media screen and (max-width: 576px) {
	.titleBtn{
		flex-basis: 100%;
	}
}


</style>