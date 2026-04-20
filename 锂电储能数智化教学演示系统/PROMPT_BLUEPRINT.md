# Project Blueprint: Lithium Battery ESS Digital Twin Teaching Dashboard

You are an expert Frontend Engineer and Product Designer. Your goal is to recreate a high-fidelity, industrial-style digital twin monitoring dashboard for a Lithium-Ion Energy Storage System (ESS) based on the following specifications.

## 1. CORE CONSTRAINTS (CRITICAL)
- **Viewport Locking**: The application MUST be strictly contained within `100vw` and `100vh`. 
- **No Scrolling**: Use `overflow: hidden` on `html`, `body`, and the main container. Zero horizontal or vertical scrollbars.
- **Box Sizing**: All elements must use `box-sizing: border-box` to ensure padding/borders do not break the 100% dimensions.
- **Layout Precision**: Use Flexbox with `min-h-0` on flexible sections to ensure charts and lists shrink to fit the viewport instead of pushing content out.

## 2. VISUAL IDENTITY & THEME
- **Color Palette**: 
  - Background: Deep Navy/Slate (`#020617`).
  - Panel Background: Translucent Dark (`rgba(7, 12, 28, 0.85)`).
  - Accents: Neon Cyan (`#00f2ff`), Neon Orange (`#ffaa00`), Neon Red (`#ff0055`), Neon Green (`#00ff9d`).
- **Aesthetic**: "Industrial Cyberpunk". Use corner accents, technical scanlines, and glow effects.
- **Typography**: Orbitron for headings/digital numbers, JetBrains Mono for technical data/logs, Noto Sans SC for main text.

## 3. COMPONENT ARCHITECTURE
- **Header**: 16rem height. Contains Platform Name, Status Light (Success/Warning/Danger), and a striking Digital Clock (Skewed/Trapezoid style).
- **KPI Grid**: 4 top-level cards (SOC, Power, Energy, Temperature) with color-coded side borders and mini-sparklines or progress bars.
- **Visualization Area**:
  - **Gauge Chart**: Real-time BMS voltage monitoring.
  - **Trend Chart**: Dual-axis line chart showing Power (kW) vs. SOC (%) correlation.
- **AI Control Hub (Sidebar)**:
  - Scenario Selector: Toggle between 'Night Charging', 'Noon Peak', and 'Imbalance Warning'.
  - AI Analysis Box: Live textual explanation of the current state.
  - JSON Data Bridge: A read-only textarea showing the live "raw data stream".
- **Startup Action Panel**: Large status indicator with a dynamic battery icon.

## 4. DYNAMIC BATTERY ICON BEHAVIOR
Implement a custom CSS battery icon that changes based on system state:
1. **Charging (Success)**: Width mapped to SOC, Cyan color, with a flowing diagonal light overlay (`animate-flow`).
2. **Warning (High Temp)**: Orange color, breathing pulse glow effect (`animate-shell-glow`).
3. **Danger (Imbalance)**: Red color, high-frequency physical shake animation (`animate-shake`).

## 5. SCENARIO LOGIC (STATE MACHINE)
- **Scenario 1 (Night)**: Time=23:30, Price=Valley, Logic="Low Price + Normal Temp", Action="Valley Charging".
- **Scenario 2 (Noon)**: Time=12:40, Price=Peak, Status="Warning", Logic="Peak Price + High Ambient Temp", Action="Discharge Mode".
- **Scenario 3 (Alert)**: Time=15:20, Status="Danger", Logic="Voltage Imbalance > 5℃ detected by AI", Action="Safety Intervention".

## 6. CSS RECIPES (Tailwind/CSS)
- **Panel Clip-path**: `polygon(0 0, calc(100% - 10px) 0, 100% 10px, 100% 100%, 10px 100%, 0 calc(100% - 10px))`
- **Time Box Style**: SkewX(-5deg), border-2 cyan, trapezoid clip-path.
- **Neon Text Shadow**: `text-shadow: 0 0 12px var(--neon-cyan)`.
- **Scanline**: Linear gradient with 4px repeat size across the background.

## 7. TECH STACK REQUIREMENTS
- **Framework**: Vue 3 (Reactive State).
- **Charts**: Apache ECharts (Canvas/SVG).
- **Styling**: Tailwind CSS 4.x.
- **Icons**: Lucide for technical symbols (Zap, Cpu, AlertTriangle).

---
**PROMPT USAGE**: Paste this blueprint into a code-generation agent to build or iterate upon this specific ESS Digital Twin interface.
