<script>
	import TailwindCss from './TailwindCSS.svelte';
	import { Line } from 'svelte-chartjs';
	import { Chart, Title, Tooltip, Legend, BarElement, CategoryScale, LineElement, LinearScale, PointElement } from 'chart.js';
	// import Nouislider from '@jbfulgencio/svelte-nouislider';
	Chart.register(Title, Tooltip, Legend, BarElement, CategoryScale, LineElement, LinearScale, PointElement);
	import { onMount, afterUpdate } from 'svelte';
	import LevelSliders from './LevelSliders.svelte';
	import Slider from './Slider.svelte';

	var hostIP = '192.168.168.171';

	var windowWidth, windowHeight;

	const socket = new WebSocket(`ws://${hostIP}:8080`);

	var state = {};

	const displayLength = 100;

	var wsData = {};
	var myriadData = [];
	// Connection opened
	socket.addEventListener('open', (event) => {
		getMyriad('settings');
	});

	function getMyriad(req = 'status') {
		socket.send(JSON.stringify({ request: req }));
	}

	// Listen for messages
	socket.addEventListener('message', (event) => {
		var rawData = JSON.parse(event.data);
		Object.keys(rawData).forEach((key) => {
			if (key.indexOf('cur') != -1 && rawData[key] == 0) {
				console.log('Unexpected value received');
			} else {
				wsData[key] = rawData[key];
			}
			if (rawData.hasOwnProperty('mute_switch')) {
				state['mute'] = wsData.mute_switch == 1 ? true : false;
			}
			if (rawData.hasOwnProperty('smvol_switch')) {
				state['smvol'] = wsData.smvol_switch == 1 ? true : false;
			}
			if (rawData.hasOwnProperty('volpot_switch')) {
				state['volpot'] = wsData.volpot_switch == 1 ? true : false;
			}
			if (rawData.hasOwnProperty('level_min')) {
				state['level_min'] = roundNumber(wsData.level_min / 2.55, 0);
			}
			if (rawData.hasOwnProperty('level_max')) {
				state['level_max'] = roundNumber(wsData.level_max / 2.55, 0);
			}
			if (rawData.hasOwnProperty('smvol_register')) {
				state['smvol_register'] = wsData.smvol_register;
			}
			if (rawData.hasOwnProperty('response_time')) {
				state['response_time'] = wsData.response_time;
			}
		});
		var statusItems = [wsData.cur_mic, wsData.cur_amb, wsData.cur_input, wsData.cur_output, wsData.volpot_register];
		wsData = Object.assign(wsData, JSON.parse(event.data));
		for (var i = 0; i < statusItems.length; i++) {
			var addVal = i == 4 ? statusItems[i] / 40.95 : Number(statusItems[i]);
			myriadChartData.datasets[i].data = [...myriadChartData.datasets[i].data, i == 4 ? statusItems[i] / 40.95 : statusItems[i] / 2];
			if (myriadChartData.datasets[i].data.length > displayLength) {
				myriadChartData.datasets[i].data.shift();
			}
		}
	});

	var myriadChartData = {
		labels: [],
		datasets: [
			{
				label: 'Microphone Level',
				fill: true,
				data: [],
				backgroundColor: 'rgba(250, 31, 250, 0.8)',
				borderColor: 'rgba(250, 31, 250, 0.8)',
				borderWidth: 1,
				pointRadius: 0,
			},
			{
				label: 'Ambient Level',
				data: [],
				backgroundColor: 'rgba(245, 40, 145, 0.8)',
				borderColor: 'rgba(245, 40, 145, 0.8)',
				borderWidth: 1,
				pointRadius: 0,
			},
			{
				label: 'Audio Input',
				data: [],
				backgroundColor: 'rgba(56, 245, 39, 0.8)',
				borderColor: 'rgba(56, 245, 39, 0.8)',
				borderWidth: 1,
				pointRadius: 0,
			},
			{
				label: 'Audio Output',
				data: [],
				backgroundColor: 'rgba(245, 176, 39, 0.8)',
				borderColor: 'rgba(245, 176, 39, 0.8)',
				borderWidth: 1,
				pointRadius: 0,
			},
			{
				label: 'Volume Level',
				data: [],
				backgroundColor: 'rgba(245, 39, 39, 0.8)',
				borderColor: 'rgba(245, 39, 39, 0.8)',
				borderWidth: 1,
				pointRadius: 0,
			},
		],
	};

	function sendCommand(cmd) {
		socket.send(cmd);
	}

	function toggleMute() {
		socket.send(JSON.stringify({ set: state.mute ? 'mute_on' : 'mute_off' }));
	}

	function toggleSMVol() {
		socket.send(JSON.stringify({ set: state.smvol ? 'smvol_on' : 'smvol_off' }));
	}

	function toggleVolPot() {
		socket.send(JSON.stringify({ set: state.volpot ? 'volpot_on' : 'volpot_off' }));
	}

	function myriadLoop() {
		setTimeout(() => {
			getMyriad(), myriadLoop();
		}, 100);
	}

	for (let i = 0; i < displayLength; i++) {
		myriadChartData.labels[i] = i;
		for (let j = 0; j < myriadChartData.datasets.length; j++) {
			myriadChartData.datasets[j].data[i] = null;
		}
	}

	function checkMinLevel() {
		const minInput = document.querySelector('#input-level-min');
		const maxInput = document.querySelector('#input-level-max');
		if (Number(minInput.value) >= Number(maxInput.value)) {
			minInput.value = maxInput.value;
			state.level_min = minInput.value;
		}
	}

	function checkMaxLevel() {
		const minInput = document.querySelector('#input-level-min');
		const maxInput = document.querySelector('#input-level-max');
		if (Number(minInput.value) >= Number(maxInput.value)) {
			maxInput.value = minInput.value;
			state.level_max = maxInput.value;
		}
	}

	function sendLevelMin() {
		socket.send(JSON.stringify({ set: 'levelmin_' + roundNumber(state.level_min * 2.55, 0) }));
	}

	function sendLevelMax() {
		socket.send(JSON.stringify({ set: 'levelmax_' + roundNumber(state.level_max * 2.55, 0) }));
	}

	function setLevels(event) {
		if (state.level_min != event.detail.values[0]) {
			state.level_min = event.detail.values[0];
			sendLevelMin();
		}
		if (state.level_max != event.detail.values[1]) {
			state.level_max = event.detail.values[1];
			sendLevelMax();
		}
	}

	function setSMVol(event) {
		if (state.smvol_register != event.detail.values) {
			state.smvol_register = event.detail.values;
			sendSMVol();
		}
	}

	function sendSMVol() {
		socket.send(JSON.stringify({ set: 'smvol_' + state.smvol_register }));
	}

	function setResponseTime(event) {
		if (state.response_time != event.detail.values) {
			state.response_time = event.detail.values;
			sendResponseTime();
		}
	}

	function sendResponseTime() {
		socket.send(JSON.stringify({ set: 'responsetime_' + state.response_time }));
	}

	myriadLoop();

	function roundNumber(rnum, rlength) {
		var newnumber = Math.round(rnum * Math.pow(10, rlength)) / Math.pow(10, rlength);
		return newnumber;
	}

</script>

<svelte:window bind:innerHeight={windowHeight} bind:innerWidth={windowWidth} />

<TailwindCss />
<div class="container grid grid-rows-8 w-full h-[60rem]">
	<!-- <p class="text-5xl">Brownie!</p>
	<p class="text-xl">A better way to interact with Brown Myriad amplifiers.</p> -->
	<div class="grid grid-cols-12 row-span-7">
		<div class="col-span-11">
			<Line
				data={myriadChartData}
				updateMode={'none'}
				options={({ animation: { duration: 0 } },
				{
					scales: {
						x: { ticks: { display: false } },
						ydBV: {
							title: {
								display: true,
								text: 'dBV',
								align: 'end',
							},
							type: 'linear',
							max: 100,
							min: 0,
							ticks: {
								min: 0,
								max: 100,
								stepSize: 12.5,
								callback: (value) => value * 0.8 - 70,
							},
						},
						ydBSPL: {
							title: {
								display: true,
								text: 'dB SPL',
								align: 'end',
							},
							type: 'linear',
							max: 100,
							min: 0,
							position: 'right',
							gridLines: {
								display: false,
							},
							ticks: {
								min: 0,
								max: 100,
								stepSize: 12.5,
								callback: (value) => value * 0.8 + 35,
							},
						},
					},
				})}
			/>
		</div>
		<div class="flex">
			<div class="h-full flex inputs-container items-start">
				<div id="levels-slider" />
				<div class="flex p-4 h-full">
					<LevelSliders levelMin={state.level_min} levelMax={state.level_max} on:change={setLevels} />
				</div>
				<div class="flex level-inputs-container w-32 text-right">
					<div class="self-start absolute">
						<p class="text-md w-full">Max level</p>
						<input type="number" class="w-12" id="input-level-max" max="100" min="0" bind:value={state.level_max} on:input={checkMaxLevel} on:change={sendLevelMax} />
					</div>
					<div class="self-center">
						<p class="text-xs text-justify w-16 mt-8">These values represent the loudest and quitest possible output, not absolute levels.</p>
					</div>
					<div class="self-end absolute text-right">
						<p class="text-md w-full">Min level</p>
						<input type="number" class="w-12 justify-self-end" id="input-level-min" max="100" min="0" bind:value={state.level_min} on:input={checkMinLevel} on:change={sendLevelMin} />
					</div>
				</div>
			</div>
		</div>
	</div>
	<div class="">
		<div class="grid grid-cols-2 gap-x-10 justify-center content-center my-4 w-full smvol-response-inputs">
			<div class="text-start flex-col justify-items-center">
				<div class="grid grid-cols-2 w-100 justify-items-start">
					<p class="">Smart Volume Offset</p>
					<p class="justify-self-end">{roundNumber((state.smvol_register / 255) * 40.2 - 20.1, 1)} dB</p>
				</div>
				<Slider register={state.smvol_register} on:change={setSMVol} />
				<p class="text-sm">The Smart Volume Offset adjusts the Audio Output to be louder or quieter than the Ambient Level.</p>
			</div>
			<div class="text-start flex-col justify-items-center">
				<div class="grid grid-cols-2 w-100 justify-items-start">
					<p class="">Ambient Response Time</p>
					<p class="justify-self-end">{state.response_time / 10} dB/sec</p>
				</div>
				<Slider register={state.response_time} on:change={setResponseTime} />
				<p class="text-sm">The Ambient Response Time measures how quickly the Ambient Level will match the Microphone Level.</p>
			</div>
		</div>
		<div class="flex justify-center my-4">
			<p class="text-xl mx-3">Mute: <input type="checkbox" class="form-checkbox rounded" bind:checked={state.mute} on:change={toggleMute} /></p>
			<p class="text-xl mx-3">Smart Volume: <input type="checkbox" class="form-checkbox rounded" bind:checked={state.smvol} on:change={toggleSMVol} /></p>
			<p class="text-xl mx-3">Enable Volume Knob: <input type="checkbox" class="form-checkbox rounded" bind:checked={state.volpot} on:change={toggleVolPot} /></p>
		</div>
	</div>
</div>

<style>
	.level-inputs-container {
		margin-top: 28px;
		margin-left: 10px;
		height: 90%;
	}
	.container {
		margin: 0 auto;
	}
	.smvol-response-inputs {
		margin-left: 14px;
		padding: 0 20px;
	}
</style>
