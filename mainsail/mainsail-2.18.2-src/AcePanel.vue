<template>
    <v-card class="ace-panel mb-4 shadow-sm" outlined :style="!isConnected ? 'border-color: rgba(255,255,255,0.1)' : ''">
        <v-toolbar flat height="44" dark :color="isConnected ? '' : 'grey darken-4'">
            <v-icon class="mr-2">{{ icons.multicast }}</v-icon>
            <v-toolbar-title class="text-h6 font-weight-regular">Ace Pro</v-toolbar-title>
            <v-spacer />
            <v-tooltip bottom>
                <template v-slot:activator="{ on, reset }">
                    <v-btn v-bind="reset" v-on="on" text x-small color="primary" @click="resetIndex" :disabled="disableHardwareMoves">
                        Reset Index
                    </v-btn>
                </template>
                <span>Click this to reset the current index back to none (-1) and filament position to splitter. <br /> Use only if an error occurs and you need to reset to start over. <br /> Make sure to pull all filament back to splitter.</span>
            </v-tooltip>
            <v-tooltip bottom>
                <template v-slot:activator="{ on, attrs }">
                    <v-icon v-bind="attrs" v-on="on" :color="isConnected ? 'success' : 'grey'" class="mr-3">
                        {{ isConnected ? icons.connected : icons.disconnected }}
                    </v-icon>
                </template>
                <span>{{ isConnected ? 'ACE Pro: Ready' : 'ACE Pro: Busy / Offline' }}</span>
            </v-tooltip>

            <v-btn icon small @click="refreshSlots" :disabled="!isConnected">
                <v-icon>{{ icons.refresh }}</v-icon>
            </v-btn>
        </v-toolbar>

        <v-card-text class="pa-2 position-relative">
            <div v-if="!isConnected" class="busy-lock-overlay"></div>

            <v-row no-gutters class="ace-info-row mb-2 py-1">
                <v-col cols="6" class="text-center border-right">
                    <div class="text-body-2 grey--text">Model: {{ aceModel }}</div>
                </v-col>
                <v-col cols="6" class="text-center">
                    <div class="text-body-2 grey--text">Firmware: {{ aceFirmware }}</div>
                </v-col>
            </v-row>

            <v-divider class="mb-3 mx-1" />

            <v-row no-gutters class="mb-2" :style="!isConnected ? 'opacity: 0.4; filter: grayscale(1);' : ''">
                <v-col v-for="(slot, index) in inventory" :key="index" cols="3" class="px-1">
                    <div class="slot-wrapper" :class="isConnected ? 'clickable-slot' : ''" @click="openEditDialog(index, slot)">
                        <div class="text-center mb-1">
                            <div class="text-caption grey--text text--darken-1 leading-tight">SLOT {{ index }}</div>
                            <div class="text-overline leading-tight pt-1" :class="getSlotStatusClass(slot, index)" style="font-size: 0.65rem !important; font-weight: 600;">
                                {{ getSlotStatusText(slot, index) }}
                            </div>
                        </div>

                        <div class="spool-icon-container" :class="{'active-feed': isConnected && aceData?.feed_assist_slot === index}">
                            <div class="spool-outer" :style="slot.type === 'empty' ? 'border-style: dashed; opacity: 0.4;' : ''">
                                <div class="spool-filament" :style="{ backgroundColor: formatRgb(slot.color) }"></div>
                                <div class="spool-inner"></div>
                            </div>
                        </div>

                        <div class="text-center mt-1">
                            <div class="text-caption leading-tight">{{ slot.type === 'empty' ? '---' : (slot.type || '???') }}</div>
                            <div class="text-body-2 mt-n1">{{ slot.type === 'empty' ? '--' : slot.temp }}°C</div>
                        </div>
                    </div>
                </v-col>
            </v-row>

            <div class="d-flex justify-center mb-3">
                <v-chip x-small outlined :color="isConnected ? 'primary' : 'grey'" class="pos-chip-mid">
                    <v-icon x-small left>{{ icons.position }}</v-icon>
                    Position: {{ filamentPosition.toUpperCase() }}
                </v-chip>
            </div>

            <v-divider class="my-3" />

            <v-row no-gutters class="align-center py-1" :style="!isConnected ? 'opacity: 0.4;' : ''">
                <v-col cols="4" class="text-center">
                    <div class="mb-2">
                        <div class="text-caption grey--text uppercase leading-tight">Chamber</div>
                        <div class="text-body-1">{{ currentTemp }}°C</div>
                    </div>
                    <div>
                        <div class="text-caption grey--text uppercase leading-tight">Rem. Time</div>
                        <div class="text-body-1">{{ formatTime(dryer.remain_time) }}</div>
                    </div>
                </v-col>

                <v-col cols="4" class="text-center">
                    <v-chip label small :color="isDrying ? 'orange' : 'grey darken-3'" text-color="white" class="status-chip-mid" :class="{ 'pulse-slow': isDrying }">
                        {{ isDrying ? 'DRYING' : 'IDLE' }}
                    </v-chip>
                </v-col>

                <v-col cols="4" class="text-center px-1">
                    <v-select v-model="targetTemp" :items="tempOptions" label="Target" suffix="°C" dense outlined hide-details class="compact-input centered-select mb-2" :disabled="!isConnected" />
                    <v-select v-model="selectedDuration" :items="durationOptions" label="Duration" suffix="min" dense outlined hide-details class="compact-input centered-select" :disabled="!isConnected" />
                </v-col>
            </v-row>
        </v-card-text>

        <v-card-actions class="pa-2">
            <v-btn block depressed :color="btnConfig.color" @click="handleDryerAction" class="text-body-2" :disabled="!isConnected">
                <v-icon left small v-if="showWarning">{{ icons.alert }}</v-icon>
                {{ btnConfig.text }}
            </v-btn>
        </v-card-actions>

        <v-dialog v-model="editDialog" max-width="380px">
            <v-card>
                <v-card-title class="text-subtitle-1 pb-2">Slot {{ activeSlotIndex }} Control</v-card-title>
                <v-card-text>
                    <v-alert v-if="isCurrentSlotActive" type="warning" dense text class="mb-3 text-caption">
                        Cannot change filament while printing
                    </v-alert>

                    <v-select v-model="editData.type" :items="['PLA', 'PETG', 'ABS', 'ASA', 'TPU', 'NYLON', 'PC', 'empty']" label="Type" dense outlined class="mb-2" @change="applyDefaultTemp" :disabled="isCurrentSlotActive" />
                    <div class="d-flex justify-space-between align-center mb-1">
                        <span class="text-body-2">Printing Temp</span>
                        <span class="primary--text text-body-1">{{ editData.temp }}°C</span>
                    </div>
                    <v-slider v-model="editData.temp" min="180" max="320" step="1" dense hide-details class="mb-4" :disabled="isCurrentSlotActive" />
                    <div class="d-flex justify-center mb-4">
                        <v-color-picker v-model="tempHex" hide-inputs flat width="300" :disabled="isCurrentSlotActive" />
                    </div>

                    <v-row dense class="mb-4">
                        <v-col cols="4"><v-text-field v-model.number="rgbInputs.r" label="R" type="number" dense outlined hide-details @input="syncHexFromRgb" :disabled="isCurrentSlotActive" /></v-col>
                        <v-col cols="4"><v-text-field v-model.number="rgbInputs.g" label="G" type="number" dense outlined hide-details @input="syncHexFromRgb" :disabled="isCurrentSlotActive" /></v-col>
                        <v-col cols="4"><v-text-field v-model.number="rgbInputs.b" label="B" type="number" dense outlined hide-details @input="syncHexFromRgb" :disabled="isCurrentSlotActive" /></v-col>
                    </v-row>

                    <v-divider class="my-4" />

                    <div class="text-overline mb-2">Hardware Actions</div>
                    <v-row dense>
                        <v-col cols="4"><v-btn block small depressed color="primary" @click="runSlotAction('LOAD')" :disabled="disableHardwareMoves">LOAD</v-btn></v-col>
                        <v-col cols="4"><v-btn block small depressed color="warning" @click="runSlotAction('PARK')" :disabled="disableHardwareMoves">PARK</v-btn></v-col>
                        <v-col cols="4"><v-btn block small depressed color="info" @click="runSlotAction('ASSIST')" :disabled="disableAssistToggles">ASSIST</v-btn></v-col>
                    </v-row>
                    <v-row dense class="mt-2">
                        <v-col cols="4"><v-btn block x-small depressed color="deep-orange" dark @click="runSlotAction('DISABLE_ASSIST')" :disabled="disableAssistToggles">DISABLE ASSIST</v-btn></v-col>
                        <v-col cols="4"><v-btn block x-small outlined color="success" @click="openMoveDialog('FEED')" :disabled="disableHardwareMoves">FEED...</v-btn></v-col>
                        <v-col cols="4"><v-btn block x-small outlined color="error" @click="openMoveDialog('RETRACT')" :disabled="disableHardwareMoves">RETRACT...</v-btn></v-col>
                    </v-row>
                </v-card-text>
                <v-card-actions>
                    <v-btn color="primary" depressed small @click="runSlotAction('UNLOAD_SPOOL')" :disabled="disableHardwareMoves">Unload Spool</v-btn>
                    <v-spacer /><v-btn text small @click="editDialog = false">Close</v-btn>
                    <v-btn color="primary" depressed small @click="saveFilament" :disabled="isCurrentSlotActive">Save</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>

        <v-dialog v-model="moveDialog" max-width="280px">
            <v-card>
                <v-card-title class="text-subtitle-2">{{ moveType }} (Slot {{ activeSlotIndex }})</v-card-title>
                <v-card-text class="pt-2">
                    <v-text-field v-model.number="moveParams.length" label="Length" suffix="mm" type="number" dense outlined class="mb-2" />
                    <v-text-field v-model.number="moveParams.speed" label="Speed" suffix="mm/s" type="number" dense outlined />
                </v-card-text>
                <v-card-actions>
                    <v-spacer /><v-btn text small @click="moveDialog = false">Cancel</v-btn>
                    <v-btn :color="moveType === 'FEED' ? 'success' : 'error'" depressed small @click="executeMove">Execute</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>

        <v-dialog v-model="confirmStopDialog" max-width="290">
            <v-card>
                <v-card-title class="text-h6">Stop Drying?</v-card-title>
                <v-card-text>This will shut off the ACE Pro heater and fans.</v-card-text>
                <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn text @click="confirmStopDialog = false">Cancel</v-btn>
                    <v-btn color="error" depressed @click="executeStopDrying">Stop Now</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
        <v-dialog v-model="resetIndexDialog" max-width="520">
            <v-card>
                <v-card-title class="text-h6">Reset Index?</v-card-title>
                <v-card-text>
                    <v-alert v-if="isAnySlotPrinting" type="warning" dense text class="mb-3 text-caption">
                        Cannot reset index while printing
                    </v-alert>
                    This will reset ace_current_index to -1 and ace_filament_pos to splitter. <br /> Use only if an error occurs and you need to reset to start over. <br /> Make sure to pull all filament back to splitter.
                </v-card-text>
                <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn text @click="resetIndexDialog = false">Cancel</v-btn>
                    <v-btn color="error" depressed @click="executeResetIndex">RESET</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </v-card>
</template>

<script lang="ts">
    import { Component, Vue, Watch } from 'vue-property-decorator'
    import { mdiMulticast, mdiRefresh, mdiAlertCircle, mdiPowerPlug, mdiPowerPlugOff, mdiMapMarkerOutline } from '@mdi/js'

    @Component
    export default class AcePanel extends Vue {
        icons = {
            multicast: mdiMulticast, refresh: mdiRefresh, alert: mdiAlertCircle,
            connected: mdiPowerPlug, disconnected: mdiPowerPlugOff, position: mdiMapMarkerOutline
        }

        // Default Fallbacks
        aceData: any = { model: 'ACE Pro', firmware: 'v1.x', temp: 0, dryer: { status: 'stop', remain_time: 0 } };
        inventory: any[] = [
            { type: 'empty', temp: 0, color: [120, 120, 120], status: 'ready' },
            { type: 'empty', temp: 0, color: [120, 120, 120], status: 'ready' },
            { type: 'empty', temp: 0, color: [120, 120, 120], status: 'ready' },
            { type: 'empty', temp: 0, color: [120, 120, 120], status: 'ready' }
        ];
        filamentPosition = 'unknown';
        aceStatusText = 'ready'; // Default to ready so initial info is visible
        
        connectionFailCount = 0; // Tracks consecutive network errors

        targetTemp = 55; selectedDuration = 240;
        tempOptions = [30, 35, 40, 45, 50, 55, 60];
        durationOptions = [30, 60, 90, 120, 150, 180, 210, 240];
        refreshTimer: any = null; showWarning = false;
        editDialog = false; activeSlotIndex = 0; tempHex = '#FF0000';
        editData = { type: 'PLA', temp: 220 }; rgbInputs = { r: 255, g: 0, b: 0 };
        confirmStopDialog = false;
        resetIndexDialog = false;
        moveDialog = false; moveType = 'FEED';
        moveParams = { length: 50, speed: 10 };

        materialDefaults: Record<string, number> = {
            'PLA': 220, 'PETG': 250, 'ABS': 250, 'ASA': 260, 'TPU': 230, 'NYLON': 250, 'PC': 280
        };

        get isConnected() {
            const status = this.aceStatusText?.toLowerCase() || '';
            // Added 'busy' and 'printing' so the UI doesn't visually crash if Klipper reports those
            return ['ready', 'drying', 'busy', 'printing'].includes(status);
        }
        
        get aceModel() { return this.aceData?.model || 'ACE Pro'; }
        get aceFirmware() { return this.aceData?.firmware || '---'; }
        get currentTemp() { return this.aceData?.temp ?? 0; }
        get dryer() { return this.aceData?.dryer || { status: 'stop', remain_time: 0 }; }
        get isDrying() { return this.dryer.status !== 'stop' && this.dryer.status !== undefined; }

        get btnConfig() {
            if (this.showWarning) return { color: 'warning', text: 'Target Low' };
            if (this.isDrying) return { color: 'error', text: 'STOP DRYING' };
            return { color: 'primary', text: 'START DRYING' };
        }

        get isAnySlotPrinting() {
            const slot = this.aceData?.feed_assist_slot;
            return slot !== undefined && slot !== null && slot >= 0 && slot <= 3;
        }

        get isCurrentSlotActive() {
            return this.aceData?.feed_assist_slot === this.activeSlotIndex;
        }

        get disableHardwareMoves() {
            // Disable Load/Park/Feed/Retract completely if ANY slot is printing
            return this.isAnySlotPrinting;
        }

        get disableAssistToggles() {
            // If nothing is printing, allow it everywhere
            if (!this.isAnySlotPrinting) return false;
            // If something IS printing, only allow it on the actively printing slot
            return !this.isCurrentSlotActive;
        }

        @Watch('tempHex')
        onHexChange(hex: string) {
            if (hex?.startsWith('#')) {
                this.rgbInputs.r = parseInt(hex.slice(1, 3), 16);
                this.rgbInputs.g = parseInt(hex.slice(3, 5), 16);
                this.rgbInputs.b = parseInt(hex.slice(5, 7), 16);
            }
        }

        syncHexFromRgb() {
            const r = Math.max(0, Math.min(255, this.rgbInputs.r || 0));
            const g = Math.max(0, Math.min(255, this.rgbInputs.g || 0));
            const b = Math.max(0, Math.min(255, this.rgbInputs.b || 0));
            this.tempHex = `#${((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1)}`;
        }

        async mounted() {
            await this.fetchAceData();
            this.refreshTimer = setInterval(this.fetchAceData, 2000);
        }

        beforeDestroy() { if (this.refreshTimer) clearInterval(this.refreshTimer); }

        async fetchAceData() {
            try {
                const response = await fetch('/printer/objects/query?ace&save_variables');
                if (!response.ok) throw new Error('Bad network response'); // Catch 502/503 errors
                
                const json = await response.json();

                // If we get here, the request was successful
                this.connectionFailCount = 0;

                if (!json?.result?.status) return;

                const status = json.result.status;

                // Fixed the logic here to not overwrite status with 'busy' if the object is empty
                if (status.ace) {
                    if (Object.keys(status.ace).length > 0) {
                        this.aceData = { ...this.aceData, ...status.ace };
                        // Only update status text if Klipper explicitly provided it in this payload
                        if (status.ace.status) {
                            this.aceStatusText = status.ace.status;
                        }
                    }
                }

                const vars = status.save_variables?.variables;
                if (vars) {
                    if (Array.isArray(vars.ace_inventory) && vars.ace_inventory.length === 4) {
                        this.inventory = [...vars.ace_inventory];
                    }
                    if (vars.ace_filament_pos) {
                        this.filamentPosition = vars.ace_filament_pos;
                    }
                }
            } catch (e) {
                // Increment fail counter instead of immediately going offline
                this.connectionFailCount++;
                if (this.connectionFailCount >= 3) {
                    this.aceStatusText = 'offline';
                }
            }
        }

        getSlotStatusText(slot: any, index: number) {
            if (this.aceData?.feed_assist_slot === index) return 'PRINTING';
            if (slot.type === 'empty') return 'EMPTY';
            return 'READY';
        }

        getSlotStatusClass(slot: any, index: number) {
            if (this.aceData?.feed_assist_slot === index) return 'orange--text';
            if (slot.type === 'empty') return 'grey--text text--lighten-2';
            return 'success--text';
        }

        formatRgb(rgb: any) {
            if (Array.isArray(rgb)) return `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`;
            return rgb || '#444';
        }

        async runSlotAction(action: string) {
            if (!this.isConnected) return;

            // Enforce action locks based on printing state
            if (['LOAD', 'PARK', 'UNLOAD_SPOOL'].includes(action) && this.disableHardwareMoves) return;
            if (['ASSIST', 'DISABLE_ASSIST'].includes(action) && this.disableAssistToggles) return;

            const idx = this.activeSlotIndex;
            if (action === 'LOAD') await this.sendGcode(`ACE_CHANGE_TOOL Tool=${idx}`);
            if (action === 'PARK') await this.sendGcode(`PARK_TO_TOOLHEAD INDEX=${idx}`);
            if (action === 'ASSIST') await this.sendGcode(`ACE_ENABLE_FEED_ASSIST INDEX=${idx}`);
            if (action === 'DISABLE_ASSIST') await this.sendGcode(`ACE_DISABLE_FEED_ASSIST INDEX=${idx}`);
            if (action === 'UNLOAD_SPOOL') await this.sendGcode(`ACE_CHANGE_SPOOL INDEX=${idx}`);
            this.editDialog = false;
        }

        openMoveDialog(type: string) { 
            if (this.disableHardwareMoves) return;
            this.moveType = type; 
            this.moveDialog = true; 
        }

        async executeMove() {
            if (this.disableHardwareMoves) return;
            const idx = this.activeSlotIndex;
            const { length, speed } = this.moveParams;
            const cmd = this.moveType === 'FEED' ? 'ACE_FEED' : 'ACE_RETRACT';
            await this.sendGcode(`${cmd} INDEX=${idx} LENGTH=${length} SPEED=${speed}`);
            this.moveDialog = false;
        }

        openEditDialog(index: number, slot: any) {
            if (!this.isConnected) return;
            
            this.activeSlotIndex = index;
            
            this.editData = { type: slot.type || 'PLA', temp: slot.temp || 220 };
            const rgb = slot.color || [255, 0, 0];
            this.rgbInputs = { r: rgb[0], g: rgb[1], b: rgb[2] };
            this.tempHex = `#${((1 << 24) + (rgb[0] << 16) + (rgb[1] << 8) + rgb[2]).toString(16).slice(1)}`;
            this.editDialog = true;
        }

        async saveFilament() {
            if (!this.isConnected || this.isCurrentSlotActive) return;
            
            const colorStr = `"${this.rgbInputs.r},${this.rgbInputs.g},${this.rgbInputs.b}"`;
            await this.sendGcode(`SET_SLOT INDEX=${this.activeSlotIndex} TYPE=${this.editData.type} TEMP=${this.editData.temp} COLOR=${colorStr}`);
            this.editDialog = false;
        }

        async resetIndex() {
            if (!this.isConnected) return;
            if (!this.isAnySlotPrinting) { this.resetIndexDialog = true; return; }
        }

        async executeResetIndex() {
            await this.sendGcode('RESET_INDEX');
            this.resetIndexDialog = false;
        }

        async handleDryerAction() {
            if (!this.isConnected) return;
            if (this.isDrying) { this.confirmStopDialog = true; return; }
            await this.sendGcode(`ACE_START_DRYING TEMP=${this.targetTemp} DURATION=${this.selectedDuration}`);
        }

        async executeStopDrying() {
            await this.sendGcode('ACE_STOP_DRYING');
            this.confirmStopDialog = false;
        }

        async refreshSlots() {
            if (!this.isConnected) return;
            await this.sendGcode('ACE_QUERY_SLOTS');
        }

        async sendGcode(script: string) {
            await fetch('/printer/gcode/script', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ script })
            });
        }

        formatTime(minutes: number) {
            if (!minutes || minutes <= 0) return '--:--';
            const h = Math.floor(minutes / 60); const m = Math.floor(minutes % 60);
            return h > 0 ? `${h}h ${m}m` : `${m}m`;
        }
        applyDefaultTemp(type: string) { if (this.materialDefaults[type]) this.editData.temp = this.materialDefaults[type]; }
    }
</script>

<style scoped>
    .position-relative {
        position: relative;
    }

    .busy-lock-overlay {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 10;
        background: transparent;
        cursor: not-allowed;
    }

    .leading-tight {
        line-height: 1.1;
    }

    .uppercase {
        text-transform: uppercase;
        letter-spacing: 0.6px;
        font-size: 0.65rem !important;
    }

    .compact-input {
        font-size: 0.8rem;
    }

    .centered-select :deep(input) {
        text-align: center;
    }

    .centered-select :deep(.v-select__selection) {
        width: 100%;
        justify-content: center;
    }

    .ace-info-row {
        background: rgba(255,255,255,0.03);
        border-bottom: 1px solid rgba(255,255,255,0.1);
        margin: -8px -8px 10px -8px;
    }

    .border-right {
        border-right: 1px solid rgba(255,255,255,0.1);
    }

    .pos-chip-mid {
        height: 24px !important;
        padding: 0 14px !important;
        font-size: 0.75rem !important;
    }

    .status-chip-mid {
        height: 48px !important;
        min-width: 85px;
        font-size: 0.8rem !important;
        letter-spacing: 1px;
    }

    .spool-icon-container {
        display: flex;
        justify-content: center;
        align-items: center;
        height: 68px;
    }

    .spool-outer {
        width: 56px;
        height: 56px;
        border-radius: 50%;
        border: 3px solid #333;
        position: relative;
        display: flex;
        justify-content: center;
        align-items: center;
        transition: all 0.3s ease;
    }

    .spool-filament {
        width: 100%;
        height: 100%;
        border-radius: 50%;
    }

    .spool-inner {
        width: 18px;
        height: 18px;
        border-radius: 50%;
        background-color: #1e1e1e;
        position: absolute;
        border: 2px solid #333;
    }

    .active-feed .spool-outer {
        border-color: #ff9800;
        box-shadow: 0 0 12px rgba(255,152,0,0.4);
        transform: scale(1.08);
    }

    .clickable-slot {
        cursor: pointer;
        border-radius: 4px;
        padding: 4px;
        transition: background 0.2s;
    }

    .clickable-slot:hover {
        background: rgba(255,255,255,0.08);
    }

    @keyframes pulse-bg {
        0% {
            opacity: 0.7;
        }

        50% {
            opacity: 1;
        }

        100% {
            opacity: 0.7;
        }
    }

    .pulse-slow {
        animation: pulse-bg 2s infinite ease-in-out;
    }
</style>