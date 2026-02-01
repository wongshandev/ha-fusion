<script lang="ts">
	import { states, connection, itemHeight, editMode, motion } from '$lib/Stores';
	import Icon from '@iconify/svelte';
	import { openModal } from 'svelte-modals';
	import { callService } from 'home-assistant-js-websocket';
	import { fade, fly } from 'svelte/transition';
	import { cubicOut } from 'svelte/easing';
	import Ripple from 'svelte-ripple';

	export let sel: any;

	// 视图状态: 'main' | 'controls' | 'climate'
	let currentView: 'main' | 'controls' | 'climate' = 'main';

	// Tesla 实体前缀 (如 'san')
	$: prefix = sel?.prefix || 'san';

	// 获取各种实体状态
	$: batteryEntity = $states?.[`sensor.${prefix}_battery`];
	$: rangeEntity = $states?.[`sensor.${prefix}_range`];
	$: climateEntity = $states?.[`climate.${prefix}_hvac_climate_system`];
	$: lockEntity = $states?.[`lock.${prefix}_door_lock`];
	$: frunkEntity = $states?.[`cover.${prefix}_frunk`];
	$: trunkEntity = $states?.[`cover.${prefix}_trunk`];
	$: chargePortEntity = $states?.[`cover.${prefix}_charger_door`];
	$: chargingEntity = $states?.[`binary_sensor.${prefix}_charging`];
	$: interiorTempEntity = $states?.[`sensor.${prefix}_inside_temperature`];
	$: exteriorTempEntity = $states?.[`sensor.${prefix}_outside_temperature`];
	$: seatHeaterFLEntity = $states?.[`select.${prefix}_heated_seat_front_left`];
	$: seatHeaterFREntity = $states?.[`select.${prefix}_heated_seat_front_right`];
	$: seatHeaterRLEntity = $states?.[`select.${prefix}_heated_seat_rear_left`];
	$: seatHeaterRCEntity = $states?.[`select.${prefix}_heated_seat_rear_center`];
	$: seatHeaterRREntity = $states?.[`select.${prefix}_heated_seat_rear_right`];

	// 计算值
	$: batteryLevel = batteryEntity?.state ? parseInt(batteryEntity.state) : 0;
	$: range = rangeEntity?.state ? parseFloat(rangeEntity.state).toFixed(1) : '0';
	$: isLocked = lockEntity?.state === 'locked';
	$: isFrunkOpen = frunkEntity?.state === 'open';
	$: isTrunkOpen = trunkEntity?.state === 'open';
	$: isCharging = chargingEntity?.state === 'on';
	$: isClimateOn = climateEntity?.state !== 'off';
	$: interiorTemp = interiorTempEntity?.state || '--';
	$: exteriorTemp = exteriorTempEntity?.state || '--';
	$: targetTemp = climateEntity?.attributes?.temperature || 20;

	// 座椅加热等级 (off, low, medium, high)
	$: seatHeaterFL = getSeatHeaterLevel(seatHeaterFLEntity?.state);
	$: seatHeaterFR = getSeatHeaterLevel(seatHeaterFREntity?.state);
	$: seatHeaterRL = getSeatHeaterLevel(seatHeaterRLEntity?.state);
	$: seatHeaterRC = getSeatHeaterLevel(seatHeaterRCEntity?.state);
	$: seatHeaterRR = getSeatHeaterLevel(seatHeaterRREntity?.state);

	function getSeatHeaterLevel(state: string | undefined): number {
		if (!state || state === 'off') return 0;
		if (state === 'low') return 1;
		if (state === 'medium') return 2;
		if (state === 'high') return 3;
		return 0;
	}

	// 图片路径
	$: carImage = getCarImage();

	function getCarImage(): string {
		const model = sel?.model || 'y';
		const color = sel?.color || 'white';
		
		if (isFrunkOpen) {
			return `/tesla/models/${model}/${color}/baseFrunkOpened.jpg`;
		} else if (isTrunkOpen) {
			return `/tesla/models/${model}/${color}/baseTrunkOpened.jpg`;
		} else if (isCharging) {
			return `/tesla/models/${model}/${color}/baseChargeportOpened.jpg`;
		}
		return `/tesla/models/${model}/${color}/baseWide.jpg`;
	}

	// 操作函数
	async function toggleLock() {
		const service = isLocked ? 'unlock' : 'lock';
		await callService($connection, 'lock', service, {
			entity_id: lockEntity?.entity_id
		});
	}

	async function openFrunk() {
		await callService($connection, 'cover', 'open_cover', {
			entity_id: frunkEntity?.entity_id
		});
	}

	async function openTrunk() {
		await callService($connection, 'cover', 'open_cover', {
			entity_id: trunkEntity?.entity_id
		});
	}

	async function toggleClimate() {
		if (isClimateOn) {
			await callService($connection, 'climate', 'turn_off', {
				entity_id: climateEntity?.entity_id
			});
		} else {
			await callService($connection, 'climate', 'turn_on', {
				entity_id: climateEntity?.entity_id
			});
		}
	}

	async function setTemperature(temp: number) {
		await callService($connection, 'climate', 'set_temperature', {
			entity_id: climateEntity?.entity_id,
			temperature: temp
		});
	}

	async function defrost() {
		await callService($connection, 'climate', 'set_preset_mode', {
			entity_id: climateEntity?.entity_id,
			preset_mode: 'Defrost'
		});
	}

	async function ventWindows() {
		await callService($connection, 'button', 'press', {
			entity_id: `button.${prefix}_vent_windows`
		});
	}

	async function cycleSeatHeater(position: string) {
		const entityId = `select.${prefix}_heated_seat_${position}`;
		const entity = $states?.[entityId];
		if (!entity) return;

		const options = entity.attributes?.options || ['off', 'low', 'medium', 'high'];
		const currentIndex = options.indexOf(entity.state);
		const nextIndex = (currentIndex + 1) % options.length;
		
		await callService($connection, 'select', 'select_option', {
			entity_id: entityId,
			option: options[nextIndex]
		});
	}

	async function handleClick() {
		if ($editMode) {
			openModal(() => import('$lib/Modal/TeslaConfig.svelte'), { sel });
		}
	}

	function goBack() {
		currentView = 'main';
	}
</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
	class="tesla-card"
	style:height="calc({$itemHeight}px * 6 + 0.4rem * 5)"
	on:click={handleClick}
>
	{#if currentView === 'main'}
		<!-- 主视图 -->
		<div class="main-view" in:fade={{ duration: $motion }}>
			<!-- 顶部信息栏 -->
			<div class="top-bar">
				<div class="range-info">
					<div class="battery-icon" class:charging={isCharging}>
						<Icon icon={isCharging ? "mdi:battery-charging" : "mdi:battery"} width="24" />
					</div>
					<span class="range-value">{range} km</span>
				</div>
				<div class="status-icons">
					<button class="icon-btn" on:click|stopPropagation={toggleLock}>
						<Icon icon={isLocked ? "mdi:lock" : "mdi:lock-open"} width="24" />
					</button>
					<button class="icon-btn" on:click|stopPropagation={toggleClimate}>
						<Icon icon="mdi:fan" width="24" class:active={isClimateOn} />
					</button>
				</div>
			</div>

			<!-- 车辆图片 -->
			<div class="car-image-container">
				<img src={carImage} alt="Tesla" class="car-image" />
			</div>

			<!-- 底部菜单 -->
			<div class="bottom-menu">
				<button 
					class="menu-item"
					on:click|stopPropagation={() => currentView = 'controls'}
				>
					<Icon icon="mdi:car-cog" width="28" />
					<span>Controls</span>
				</button>
				<button 
					class="menu-item"
					on:click|stopPropagation={() => currentView = 'climate'}
				>
					<Icon icon="mdi:thermostat" width="28" />
					<span>Climate</span>
					<span class="temp-badge">{interiorTemp} °C</span>
				</button>
			</div>
		</div>

	{:else if currentView === 'controls'}
		<!-- Controls 视图 -->
		<div class="controls-view" in:fly={{ x: 100, duration: $motion }}>
			<div class="view-header">
				<button class="back-btn" on:click|stopPropagation={goBack}>
					<Icon icon="mdi:chevron-left" width="32" />
				</button>
				<span class="view-title">Controls</span>
			</div>

			<div class="controls-grid">
				<!-- 锁车 -->
				<button 
					class="control-btn" 
					class:active={!isLocked}
					on:click|stopPropagation={toggleLock}
				>
					<Icon icon={isLocked ? "mdi:lock" : "mdi:lock-open"} width="36" />
					<span>{isLocked ? 'Locked' : 'Unlocked'}</span>
				</button>

				<!-- 前备箱 -->
				<button 
					class="control-btn"
					class:active={isFrunkOpen}
					on:click|stopPropagation={openFrunk}
				>
					<Icon icon="mdi:car-select" width="36" />
					<span>Frunk</span>
				</button>

				<!-- 后备箱 -->
				<button 
					class="control-btn"
					class:active={isTrunkOpen}
					on:click|stopPropagation={openTrunk}
				>
					<Icon icon="mdi:car-back" width="36" />
					<span>Trunk</span>
				</button>

				<!-- 空调 -->
				<button 
					class="control-btn"
					class:active={isClimateOn}
					on:click|stopPropagation={toggleClimate}
				>
					<Icon icon="mdi:fan" width="36" />
					<span>Climate</span>
				</button>

				<!-- 通风 -->
				<button 
					class="control-btn"
					on:click|stopPropagation={ventWindows}
				>
					<Icon icon="mdi:car-door" width="36" />
					<span>Vent</span>
				</button>

				<!-- 闪灯 -->
				<button 
					class="control-btn"
					on:click|stopPropagation={() => callService($connection, 'button', 'press', { entity_id: `button.${prefix}_flash_lights` })}
				>
					<Icon icon="mdi:car-light-high" width="36" />
					<span>Flash</span>
				</button>
			</div>
		</div>

	{:else if currentView === 'climate'}
		<!-- Climate 视图 -->
		<div class="climate-view" in:fly={{ x: 100, duration: $motion }}>
			<div class="view-header">
				<button class="back-btn" on:click|stopPropagation={goBack}>
					<Icon icon="mdi:chevron-left" width="32" />
				</button>
				<span class="view-title">Climate</span>
			</div>

			<!-- 温度显示 -->
			<div class="temp-display">
				<div class="temp-info">
					<span class="temp-label">Interior</span>
					<span class="temp-value">{interiorTemp} °C</span>
				</div>
				<div class="temp-divider">-</div>
				<div class="temp-info">
					<span class="temp-label">Exterior</span>
					<span class="temp-value">{exteriorTemp} °C</span>
				</div>
			</div>

			<!-- 座椅加热布局 -->
			<div class="seat-heaters">
				<div class="car-top-view">
					<!-- 前排座椅 -->
					<div class="seat-row front">
						<button 
							class="seat-btn" 
							class:active={seatHeaterFL > 0}
							on:click|stopPropagation={() => cycleSeatHeater('front_left')}
						>
							<Icon icon="mdi:car-seat-heater" width="32" />
							{#if seatHeaterFL > 0}
								<span class="heat-level">{seatHeaterFL}</span>
							{/if}
						</button>
						<button 
							class="seat-btn"
							class:active={seatHeaterFR > 0}
							on:click|stopPropagation={() => cycleSeatHeater('front_right')}
						>
							<Icon icon="mdi:car-seat-heater" width="32" />
							{#if seatHeaterFR > 0}
								<span class="heat-level">{seatHeaterFR}</span>
							{/if}
						</button>
					</div>
					<!-- 后排座椅 -->
					<div class="seat-row rear">
						<button 
							class="seat-btn"
							class:active={seatHeaterRL > 0}
							on:click|stopPropagation={() => cycleSeatHeater('rear_left')}
						>
							<Icon icon="mdi:car-seat-heater" width="24" />
							{#if seatHeaterRL > 0}
								<span class="heat-level">{seatHeaterRL}</span>
							{/if}
						</button>
						<button 
							class="seat-btn small"
							class:active={seatHeaterRC > 0}
							on:click|stopPropagation={() => cycleSeatHeater('rear_center')}
						>
							<Icon icon="mdi:car-seat-heater" width="20" />
							{#if seatHeaterRC > 0}
								<span class="heat-level">{seatHeaterRC}</span>
							{/if}
						</button>
						<button 
							class="seat-btn"
							class:active={seatHeaterRR > 0}
							on:click|stopPropagation={() => cycleSeatHeater('rear_right')}
						>
							<Icon icon="mdi:car-seat-heater" width="24" />
							{#if seatHeaterRR > 0}
								<span class="heat-level">{seatHeaterRR}</span>
							{/if}
						</button>
					</div>
				</div>
			</div>

			<!-- 温度控制 -->
			<div class="temp-control">
				<button 
					class="temp-btn"
					class:active={isClimateOn}
					on:click|stopPropagation={toggleClimate}
				>
					<Icon icon="mdi:power" width="28" />
				</button>
				<div class="target-temp">
					<button class="temp-adjust" on:click|stopPropagation={() => setTemperature(targetTemp - 0.5)}>
						<Icon icon="mdi:minus" width="24" />
					</button>
					<span class="target-value">{targetTemp}</span>
					<button class="temp-adjust" on:click|stopPropagation={() => setTemperature(targetTemp + 0.5)}>
						<Icon icon="mdi:plus" width="24" />
					</button>
				</div>
				<button class="temp-btn" on:click|stopPropagation={ventWindows}>
					<Icon icon="mdi:car-door" width="28" />
					<span>Vent</span>
				</button>
			</div>

			<!-- 除霜按钮 -->
			<div class="defrost-section">
				<button class="defrost-btn" on:click|stopPropagation={defrost}>
					<Icon icon="mdi:car-defrost-front" width="28" />
					<span>Defrost Car</span>
				</button>
			</div>
		</div>
	{/if}
</div>

<style>
	.tesla-card {
		background: linear-gradient(180deg, #1a1a2e 0%, #16213e 100%);
		border-radius: 0.65rem;
		overflow: hidden;
		position: relative;
		color: white;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
	}

	/* Main View */
	.main-view {
		height: 100%;
		display: flex;
		flex-direction: column;
	}

	.top-bar {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 1rem;
	}

	.range-info {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.battery-icon {
		color: #4caf50;
	}

	.battery-icon.charging {
		color: #2196f3;
		animation: pulse 1.5s infinite;
	}

	@keyframes pulse {
		0%, 100% { opacity: 1; }
		50% { opacity: 0.5; }
	}

	.range-value {
		font-size: 1.2rem;
		font-weight: 600;
	}

	.status-icons {
		display: flex;
		gap: 0.5rem;
	}

	.icon-btn {
		background: rgba(255, 255, 255, 0.1);
		border: none;
		border-radius: 50%;
		width: 40px;
		height: 40px;
		display: flex;
		align-items: center;
		justify-content: center;
		color: white;
		cursor: pointer;
		transition: background 0.2s;
	}

	.icon-btn:hover {
		background: rgba(255, 255, 255, 0.2);
	}

	.icon-btn :global(.active) {
		color: #2196f3;
	}

	.car-image-container {
		flex: 1;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0 1rem;
	}

	.car-image {
		max-width: 100%;
		max-height: 100%;
		object-fit: contain;
	}

	.bottom-menu {
		display: flex;
		gap: 0.5rem;
		padding: 1rem;
	}

	.menu-item {
		flex: 1;
		background: rgba(255, 255, 255, 0.1);
		border: none;
		border-radius: 0.5rem;
		padding: 0.75rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.25rem;
		color: white;
		cursor: pointer;
		transition: background 0.2s;
	}

	.menu-item:hover {
		background: rgba(255, 255, 255, 0.2);
	}

	.menu-item span {
		font-size: 0.85rem;
	}

	.temp-badge {
		font-size: 0.75rem;
		opacity: 0.7;
	}

	/* View Header */
	.view-header {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		padding: 0.75rem 1rem;
		border-bottom: 1px solid rgba(255, 255, 255, 0.1);
	}

	.back-btn {
		background: none;
		border: none;
		color: white;
		cursor: pointer;
		padding: 0;
		display: flex;
		align-items: center;
	}

	.view-title {
		font-size: 1.1rem;
		font-weight: 600;
	}

	/* Controls View */
	.controls-view {
		height: 100%;
		display: flex;
		flex-direction: column;
	}

	.controls-grid {
		flex: 1;
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 0.75rem;
		padding: 1rem;
	}

	.control-btn {
		background: rgba(255, 255, 255, 0.08);
		border: 1px solid rgba(255, 255, 255, 0.1);
		border-radius: 0.75rem;
		padding: 1rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		gap: 0.5rem;
		color: white;
		cursor: pointer;
		transition: all 0.2s;
	}

	.control-btn:hover {
		background: rgba(255, 255, 255, 0.15);
	}

	.control-btn.active {
		background: rgba(33, 150, 243, 0.3);
		border-color: #2196f3;
	}

	.control-btn span {
		font-size: 0.8rem;
	}

	/* Climate View */
	.climate-view {
		height: 100%;
		display: flex;
		flex-direction: column;
	}

	.temp-display {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 1.5rem;
		padding: 1rem;
	}

	.temp-info {
		text-align: center;
	}

	.temp-label {
		font-size: 0.75rem;
		opacity: 0.7;
	}

	.temp-value {
		display: block;
		font-size: 1.5rem;
		font-weight: 600;
	}

	.temp-divider {
		opacity: 0.5;
	}

	/* Seat Heaters */
	.seat-heaters {
		flex: 1;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0.5rem;
	}

	.car-top-view {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		background: rgba(255, 255, 255, 0.05);
		padding: 1.5rem;
		border-radius: 1rem;
	}

	.seat-row {
		display: flex;
		justify-content: center;
		gap: 1.5rem;
	}

	.seat-row.rear {
		gap: 0.75rem;
	}

	.seat-btn {
		background: rgba(255, 255, 255, 0.1);
		border: 1px solid rgba(255, 255, 255, 0.15);
		border-radius: 0.5rem;
		width: 50px;
		height: 50px;
		display: flex;
		align-items: center;
		justify-content: center;
		color: rgba(255, 255, 255, 0.6);
		cursor: pointer;
		position: relative;
		transition: all 0.2s;
	}

	.seat-btn.small {
		width: 40px;
		height: 40px;
	}

	.seat-btn:hover {
		background: rgba(255, 255, 255, 0.2);
	}

	.seat-btn.active {
		background: rgba(255, 87, 34, 0.4);
		border-color: #ff5722;
		color: #ff5722;
	}

	.heat-level {
		position: absolute;
		bottom: 2px;
		right: 4px;
		font-size: 0.65rem;
		font-weight: bold;
	}

	/* Temperature Control */
	.temp-control {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 1.5rem;
		padding: 0.75rem;
	}

	.temp-btn {
		background: rgba(255, 255, 255, 0.1);
		border: 1px solid rgba(255, 255, 255, 0.15);
		border-radius: 50%;
		width: 50px;
		height: 50px;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		color: white;
		cursor: pointer;
		transition: all 0.2s;
	}

	.temp-btn span {
		font-size: 0.6rem;
	}

	.temp-btn:hover {
		background: rgba(255, 255, 255, 0.2);
	}

	.temp-btn.active {
		background: rgba(33, 150, 243, 0.4);
		border-color: #2196f3;
	}

	.target-temp {
		display: flex;
		align-items: center;
		gap: 0.75rem;
	}

	.target-value {
		font-size: 2rem;
		font-weight: 600;
		min-width: 60px;
		text-align: center;
	}

	.temp-adjust {
		background: rgba(255, 255, 255, 0.1);
		border: none;
		border-radius: 50%;
		width: 36px;
		height: 36px;
		display: flex;
		align-items: center;
		justify-content: center;
		color: white;
		cursor: pointer;
		transition: background 0.2s;
	}

	.temp-adjust:hover {
		background: rgba(255, 255, 255, 0.25);
	}

	/* Defrost Section */
	.defrost-section {
		padding: 0.5rem 1rem 1rem;
	}

	.defrost-btn {
		width: 100%;
		background: rgba(255, 255, 255, 0.08);
		border: 1px solid rgba(255, 255, 255, 0.1);
		border-radius: 0.5rem;
		padding: 0.75rem;
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 0.5rem;
		color: white;
		cursor: pointer;
		transition: background 0.2s;
	}

	.defrost-btn:hover {
		background: rgba(255, 255, 255, 0.15);
	}

	/* Phone */
	@media all and (max-width: 768px) {
		.tesla-card {
			width: calc(100vw - 2.5rem);
		}
	}
</style>
