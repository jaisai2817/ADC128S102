import serial
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from matplotlib.widgets import Button
import matplotlib.patches as mpatches

PORT = 'COM27'       # change to your port
BAUD = 115200

try:
    ser = serial.Serial(PORT, BAUD, timeout=1)
    print(f"Connected to {PORT}")
except:
    print(f"Cannot open {PORT}")
    exit()

current_bits = [0] * 16
paused = False

fig, ax = plt.subplots(figsize=(14, 6))
fig.patch.set_facecolor('white')
plt.subplots_adjust(bottom=0.22, top=0.88)

ax.set_facecolor('white')
ax.set_xlim(-0.5, 15.5)
ax.set_ylim(-0.5, 2.0)
ax.set_title('ADC128S102 — DOUT 16-bit Digital Waveform', fontsize=13, color='black', pad=12)
ax.set_xlabel('Bit Position (15 = MSB  →  0 = LSB)', fontsize=10, color='black')
ax.set_yticks([0, 1])
ax.set_yticklabels(['LOW (0)', 'HIGH (1)'], fontsize=10, color='black')
ax.set_xticks(range(16))
ax.set_xticklabels([str(i) for i in range(15, -1, -1)], fontsize=9, color='black')
ax.tick_params(colors='black')
ax.grid(True, axis='x', color='#cccccc', linewidth=0.8, linestyle='--')
ax.axhline(0, color='black', linewidth=0.8)
ax.axhline(1, color='#aaaaaa', linewidth=0.5, linestyle='--')
for s in ax.spines.values():
    s.set_edgecolor('black')

ax.axvspan(-0.5, 3.5,  alpha=0.05, color='red')
ax.axvspan(3.5,  15.5, alpha=0.05, color='green')
ax.text(1.5, 1.78, 'Leading zeros\n[bits 15-12]', ha='center', fontsize=8, color='red')
ax.text(9.5, 1.78, '12-bit ADC result  [bits 11-0]',  ha='center', fontsize=8, color='green')

def make_wave(bits):
    xs, ys = [], []
    for i, b in enumerate(bits):
        xs += [i, i + 1]
        ys += [float(b), float(b)]
    return xs, ys

xs, ys = make_wave(current_bits)
line, = ax.plot(xs, ys, color='blue', linewidth=2.5)

vlines = []
for i in range(17):
    vl, = ax.plot([i, i], [0, 0], color='blue', linewidth=2.5)
    vlines.append(vl)

bit_texts = []
for i in range(16):
    t = ax.text(i + 0.5, 0.15, '0',
                ha='center', fontsize=11,
                fontweight='bold', color='#aaaaaa')
    bit_texts.append(t)

info = ax.text(0.5, -0.32,
               'Waiting for data...',
               transform=ax.transAxes,
               ha='center', fontsize=11,
               fontweight='bold', color='black',
               bbox=dict(boxstyle='round,pad=0.4',
                         facecolor='#f0f0f0',
                         edgecolor='black',
                         linewidth=1))

status_txt = ax.text(0.5, -0.45,
                     '● RUNNING',
                     transform=ax.transAxes,
                     ha='center', fontsize=10,
                     color='green', fontweight='bold')

legend = [mpatches.Patch(color='red',   alpha=0.3, label='Leading zeros [15:12]'),
          mpatches.Patch(color='green',  alpha=0.3, label='12-bit result [11:0]')]
ax.legend(handles=legend, loc='upper right', fontsize=8)

# ── Buttons ──
ax_stop  = plt.axes([0.35, 0.05, 0.12, 0.06])
ax_start = plt.axes([0.50, 0.05, 0.12, 0.06])
ax_clear = plt.axes([0.65, 0.05, 0.12, 0.06])

btn_stop  = Button(ax_stop,  'STOP',  color='#ffcccc', hovercolor='#ff9999')
btn_start = Button(ax_start, 'START', color='#ccffcc', hovercolor='#99ff99')
btn_clear = Button(ax_clear, 'CLEAR', color='#cce5ff', hovercolor='#99ccff')

btn_stop.label.set_fontsize(11)
btn_stop.label.set_fontweight('bold')
btn_start.label.set_fontsize(11)
btn_start.label.set_fontweight('bold')
btn_clear.label.set_fontsize(11)
btn_clear.label.set_fontweight('bold')

def stop(event):
    global paused
    paused = True
    status_txt.set_text('■ PAUSED — waveform frozen')
    status_txt.set_color('red')
    fig.canvas.draw_idle()

def start(event):
    global paused
    paused = False
    status_txt.set_text('● RUNNING')
    status_txt.set_color('green')
    fig.canvas.draw_idle()

def clear(event):
    global current_bits
    current_bits = [0] * 16
    line.set_data(*make_wave(current_bits))
    for vl in vlines:
        vl.set_data([0, 0], [0, 0])
    for t in bit_texts:
        t.set_text('0')
        t.set_color('#aaaaaa')
        t.set_y(0.15)
    info.set_text('Cleared')
    fig.canvas.draw_idle()

btn_stop.on_clicked(stop)
btn_start.on_clicked(start)
btn_clear.on_clicked(clear)

def update(frame):
    global current_bits
    if paused:
        return [line, info, status_txt] + vlines + bit_texts

    try:
        raw_line = ser.readline().decode('utf-8', errors='ignore').strip()

        if len(raw_line) == 16 and all(c in '01' for c in raw_line):
            current_bits = [int(c) for c in raw_line]

            line.set_data(*make_wave(current_bits))

            for i, vl in enumerate(vlines):
                if i == 0 or i == 16:
                    b = current_bits[0] if i == 0 else current_bits[-1]
                    vl.set_data([i, i], [0, float(b)])
                else:
                    cur = current_bits[i]
                    pre = current_bits[i - 1]
                    if cur != pre:
                        vl.set_data([i, i], [0, 1])
                    else:
                        vl.set_data([i, i], [0, 0])

            for i, (b, t) in enumerate(zip(current_bits, bit_texts)):
                t.set_text(str(b))
                t.set_color('blue' if b == 1 else '#aaaaaa')
                t.set_y(1.15 if b == 1 else 0.15)

            raw_val = int(raw_line, 2)
            volt    = round((raw_val / 4095.0) * 3.3, 3)

            info.set_text(
                f'Bits: {raw_line[:4]}  {raw_line[4:8]}  '
                f'{raw_line[8:12]}  {raw_line[12:16]}'
                f'   →   Raw: {raw_val}   Voltage: {volt:.3f} V'
            )

    except:
        pass
    return [line, info, status_txt] + vlines + bit_texts

ani = animation.FuncAnimation(
    fig, update, interval=300,
    blit=False, cache_frame_data=False)

plt.show()
ser.close()
