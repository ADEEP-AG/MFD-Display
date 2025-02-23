# MFD-Display
#This replica of MFD  (Multi Function Display) used in early 2000  
import tkinter as tk
from tkinter import ttk
import random
import math

# Create the main window
root = tk.Tk()
root.title("Su-30 MKI MFD Simulation")
root.geometry("1920x1080")  # Your screen resolution
root.resizable(False, False)

# Equal-sized frames (560px width, 800px height)
nav_frame = tk.Frame(root, bd=2, relief="sunken", width=560, height=800)
nav_frame.grid(row=1, column=0, padx=40, pady=40)
tsd_frame = tk.Frame(root, bd=2, relief="sunken", width=560, height=800)
tsd_frame.grid(row=1, column=1, padx=40, pady=40)
wsd_frame = tk.Frame(root, bd=2, relief="sunken", width=560, height=800)
wsd_frame.grid(row=1, column=2, padx=40, pady=40)

nav_frame.grid_propagate(False)
tsd_frame.grid_propagate(False)
wsd_frame.grid_propagate(False)

# Top row for excess buttons
top_frame = tk.Frame(root, bg="gray", height=40)
top_frame.grid(row=0, column=0, columnspan=3, sticky="ew")

# Status bar at the bottom
status_frame = tk.Frame(root, bg="black", height=20)
status_frame.grid(row=2, column=0, columnspan=3, sticky="ew")
status_label = tk.Label(status_frame, text="System: Nominal | Warnings: None", font=("Arial", 10), fg="white", bg="black")
status_label.pack(side=tk.LEFT, padx=5)

# Missile warning at center-top
warning_frame = tk.Frame(root, bg="yellow", width=200, height=40)
warning_frame.place(x=860, y=0)  # Center-top of window
warning_label = tk.Label(warning_frame, text="", font=("Arial", 12, "bold"), fg="red", bg="yellow")
warning_label.pack(pady=5)

# Realistic button style
button_style = {
    "font": ("Arial", 9, "bold"), "bg": "#333333", "fg": "white",
    "relief": "raised", "bd": 3, "activebackground": "#555555",
    "width": 10, "height": 2
}

# Global variables
altitude = 5000  # meters
speed = 600      # km/h
g_force = 1.0    # G
fuel = 5000      # liters
thrust = 70      # % thrust
temp = 90        # °C
hydraulics = 100 # %
heading = 0      # degrees
terrain_elev = 500  # meters
targets = []  # [x, y, id, iff, locked, type, friendly, priority]
target_locked = None
missile_path = None  # [start_x, start_y, end_x, end_y, type]
hit_marker = None  # [x, y, stage]
flares = 20
chaff = 15
decoys = 10      # Missile decoys
awacs_data = {"friendly": [], "enemy": []}
incoming_missile = None  # [x, y]
selected_loadout = "AA-12 Adder"
guidance_mode = "Radar"  # Guidance mode
missile_count = 8
bomb_count = 4
blink_state = True       # For missile warning blink
ecm_effect = 0           # ECM effectiveness (0-100%)

# ---------------------------
# Navigation/Attack Display
# ---------------------------
nav_label = tk.Label(nav_frame, text="Navigation/Attack Display", font=("Arial", 14))
nav_label.pack(pady=5)
nav_canvas = tk.Canvas(nav_frame, bg="black", width=540, height=620)  # Adjusted for button grid
nav_canvas.pack(pady=5)
nav_canvas.create_rectangle(10, 10, 530, 610, outline="green", width=2)

nav_mode = "MAP"
def update_nav_data():
    global altitude, speed, g_force, fuel, thrust, temp, hydraulics, heading, terrain_elev, flares, targets, incoming_missile
    nav_canvas.delete("nav_data", "escape_vector", "map_elements", "fuel_gauge", "compass")
    altitude += random.randint(-50, 50)
    altitude = max(terrain_elev + 100, min(15000, altitude))
    speed = max(200, min(1200, speed + random.randint(-10, 10) * (thrust / 70)))
    g_force = 1 + (speed / 600) * random.uniform(-0.5, 0.5)
    fuel -= random.randint(2, 5) * (thrust / 100)
    fuel = max(0, fuel)
    temp = 80 + (thrust / 100) * 40 + random.randint(-5, 5)
    hydraulics -= random.uniform(0, 0.2)
    hydraulics = max(50, hydraulics)
    heading = heading % 360
    terrain_elev = max(0, terrain_elev + random.randint(-20, 20))

    nav_canvas.create_rectangle(50, 50, 490, 570, outline="cyan", dash=(4, 2), tags="map_elements")
    for i in range(5):
        x, y = random.randint(60, 480), random.randint(60, 560)
        nav_canvas.create_polygon(x, y, x+20, y+20, x+40, y, fill="darkgreen", tags="map_elements")
    waypoint_x, waypoint_y = random.randint(100, 440), random.randint(100, 520)
    nav_canvas.create_oval(waypoint_x-5, waypoint_y-5, waypoint_x+5, waypoint_y+5, fill="blue", tags="map_elements")
    for target in targets:
        x, y = target[0] * 0.8 + 50, target[1] * 0.8 + 50
        color = "red" if target[3] == "Enemy" else "green"
        nav_canvas.create_oval(x-3, y-3, x+3, y+3, fill=color, tags="map_elements")

    fuel_percentage = fuel / 5000
    nav_canvas.create_rectangle(20, 580, 120, 600, fill="gray", tags="fuel_gauge")
    nav_canvas.create_rectangle(20, 580, 20 + 100 * fuel_percentage, 600, fill="green" if fuel > 1000 else "red", tags="fuel_gauge")
    nav_canvas.create_text(70, 590, text=f"Fuel: {fuel:.0f}L", fill="white", font=("Arial", 8), tags="fuel_gauge")

    compass_x, compass_y = 480, 590
    nav_canvas.create_oval(compass_x-20, compass_y-20, compass_x+20, compass_y+20, outline="white", tags="compass")
    angle = math.radians(-heading)
    nav_canvas.create_line(compass_x, compass_y, compass_x + 20 * math.cos(angle), compass_y + 20 * math.sin(angle), fill="red", tags="compass")

    if nav_mode == "MAP":
        waypoint_dist = math.sqrt((waypoint_x-270)**2 + (waypoint_y-310)**2) / 10
        eta = waypoint_dist / (speed / 3.6) / 60
        border_dist = random.randint(10, 300)
        weather = random.choice(["Clear", "Cloudy", "Rain"])
        nav_canvas.create_text(270, 50, text=f"Altitude: {altitude} m", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 80, text=f"Speed: {speed} km/h", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 110, text=f"Heading: {heading}°", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 140, text=f"Waypoint: {waypoint_dist:.1f} km", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 170, text=f"ETA: {eta:.1f} min", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 200, text=f"Border: {border_dist} km", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 230, text=f"Weather: {weather}", fill="white", font=("Arial", 10), tags="nav_data")
        target_coords = "; ".join([f"ID{t[2]}: ({t[0]//10},{t[1]//10})" for t in awacs_data["enemy"]][:2])
        nav_canvas.create_text(270, 260, text=f"Targets: {target_coords or 'None'}", fill="white", font=("Arial", 10), tags="nav_data")
        if incoming_missile:
            escape_heading = (heading + random.randint(90, 270)) % 360
            nav_canvas.create_text(270, 290, text=f"Escape: {escape_heading}°", fill="red", font=("Arial", 10), tags="escape_vector")
        status_label.config(text=f"System: {'Nominal' if fuel > 500 else 'Low Fuel'} | Warnings: {'Missile' if incoming_missile else 'None'}")
    elif nav_mode == "ENGINES":
        emergency_dist = random.randint(20, 100)
        nav_canvas.create_text(270, 50, text=f"Fuel: {fuel:.0f} L", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 80, text=f"Thrust: {thrust}%", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 110, text=f"Temp: {temp:.0f}°C", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 140, text=f"Hydraulics: {hydraulics:.1f}%", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 170, text=f"Emergency Site: {emergency_dist} km", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 200, text=f"Flares: {flares}", fill="white", font=("Arial", 10), tags="nav_data")
        nav_canvas.create_text(270, 230, text=f"Status: {'Nominal' if fuel > 500 else 'Low Fuel'}", fill="white", font=("Arial", 10), tags="nav_data")
    root.after(1000, update_nav_data)

# Navigation Buttons (3x3 Grid)
terrain_mode = False
def toggle_terrain():
    global terrain_mode
    terrain_mode = not terrain_mode
    status = "Terrain Avoidance: ON" if terrain_mode else "Terrain Avoidance: OFF"
    nav_canvas.delete("terrain_status")
    nav_canvas.create_text(270, 320, text=status, fill="yellow", font=("Arial", 10), tags="terrain_status")

def switch_nav_mode():
    global nav_mode
    nav_mode = "ENGINES" if nav_mode == "MAP" else "MAP"
    nav_canvas.delete("mode_status")
    nav_canvas.create_text(270, 350, text=f"Mode: {nav_mode}", fill="yellow", font=("Arial", 10), tags="mode_status")

def adjust_heading():
    global heading
    heading = (heading + 45) % 360
    nav_canvas.delete("heading_status")
    nav_canvas.create_text(270, 380, text=f"New Heading: {heading}°", fill="yellow", font=("Arial", 10), tags="heading_status")

def update_throttle(val):
    global thrust
    thrust = int(float(val))
    nav_canvas.delete("throttle_status")
    nav_canvas.create_text(270, 410, text=f"Throttle: {thrust}%", fill="yellow", font=("Arial", 10), tags="throttle_status")

nav_button_frame = tk.Frame(nav_frame, bg="gray")
nav_button_frame.pack(side=tk.BOTTOM, pady=5)
tk.Button(nav_button_frame, text="TERRAIN", command=toggle_terrain, **button_style).grid(row=0, column=0, padx=5, pady=5)
tk.Button(nav_button_frame, text="MODE", command=switch_nav_mode, **button_style).grid(row=0, column=1, padx=5, pady=5)
tk.Button(nav_button_frame, text="HEADING", command=adjust_heading, **button_style).grid(row=0, column=2, padx=5, pady=5)
throttle = ttk.Scale(nav_button_frame, from_=50, to=100, orient=tk.VERTICAL, length=100, command=update_throttle)
throttle.set(70)
throttle.grid(row=1, column=1, pady=5)  # Centered in 3x3 grid

# ---------------------------
# Tactical Situation Display
# ---------------------------
tsd_label = tk.Label(tsd_frame, text="Tactical Situation Display", font=("Arial", 14))
tsd_label.pack(pady=5)
tsd_canvas = tk.Canvas(tsd_frame, bg="darkblue", width=540, height=620)
tsd_canvas.pack(pady=5)
center_x, center_y = 270, 310
tsd_canvas.create_oval(center_x-150, center_y-150, center_x+150, center_y+150, outline="lime", width=2, tags="radar_range")
tsd_canvas.create_oval(center_x-80, center_y-80, center_x+80, center_y+80, outline="orange", dash=(4, 2), tags="radar_range")

tsd_mode = "RADAR"
radar_mode = "Air-to-Air"
radar_range = 100
jamming_active = False
def update_radar():
    global targets, target_locked, missile_path, hit_marker, flares, chaff, decoys, incoming_missile, awacs_data, radar_range, blink_state, ecm_effect
    tsd_canvas.delete("targets", "tsd_status", "missile_path", "hit_marker")
    blink_state = not blink_state

    range_size = radar_range * 1.5
    tsd_canvas.coords("radar_range", center_x-range_size, center_y-range_size, center_x+range_size, center_y+range_size)

    if len(targets) < 6 and random.random() < 0.05:
        x, y = random.randint(60, 480), random.randint(60, 560)
        target_id = len(targets) + 1
        iff = random.choice(["Enemy"] * 4 + ["Friendly"] * 2)
        target_type = random.choice(["Air", "SAM"] if iff == "Enemy" else ["Air"])
        priority = random.randint(1, 5) if iff == "Enemy" and awacs_data else 0
        targets.append([x, y, target_id, iff, False, target_type, iff == "Friendly", priority])
        if iff == "Enemy" and random.random() < 0.1:
            incoming_missile = [x, y]

    awacs_data["friendly"] = [[t[0], t[1], t[2]] for t in targets if t[6]]
    awacs_data["enemy"] = [[t[0], t[1], t[2]] for t in targets if not t[6]]
    for target in targets:
        if target[5] == "SAM" and target[6]:
            if random.random() < 0.2:
                target[7] = random.randint(1, 5)

    if incoming_missile:
        ix, iy = incoming_missile
        dist = math.sqrt((ix - center_x)**2 + (iy - center_y)**2) / 10
        warning_label.config(text=f"Missile: {dist:.1f} km", fg="red" if blink_state else "white")
        ix += (center_x - ix) // 10
        iy += (center_y - iy) // 10
        incoming_missile = [ix, iy]
        tsd_canvas.create_line(ix, iy, center_x, center_y, fill="red", dash=(4, 2), tags="missile_path")
        if math.sqrt((ix - center_x)**2 + (iy - center_y)**2) < 20:
            incoming_missile = None
            warning_label.config(text="")
    else:
        warning_label.config(text="")

    ecm_effect = min(100, 20 + (50 if jamming_active else 0))

    if tsd_mode == "RADAR":
        for target in targets[:]:
            x, y, target_id, iff, locked, target_type, friendly, priority = target
            if radar_mode == "Air-to-Air" and target_type == "Air":
                dx, dy = random.randint(-5, 5), random.randint(-5, 5)
                x += dx
                y += dy
            dist = math.sqrt((x - center_x)**2 + (y - center_y)**2) / 10
            bearing = math.degrees(math.atan2(y - center_y, x - center_x)) % 360
            if dist > radar_range * (1 - ecm_effect / 200):
                targets.remove(target)
                if target_locked == target_id:
                    target_locked = None
                continue
            color = "red" if iff == "Enemy" else "green"
            if locked:
                color = "yellow"
            shape = "oval" if target_type == "Air" else "rectangle"
            if shape == "oval":
                tsd_canvas.create_oval(x-5, y-5, x+5, y+5, fill=color, tags="targets")
            else:
                tsd_canvas.create_rectangle(x-5, y-5, x+5, y+5, fill=color, tags="targets")
            if locked:
                tsd_canvas.create_rectangle(x-10, y-10, x+10, y+10, outline="yellow", tags="targets")
            text_x = min(max(x, center_x-200), center_x+200)
            text_y = min(max(y-15, center_y-200), center_y+200)
            tsd_canvas.create_text(text_x, text_y, text=f"ID:{target_id} {iff}", fill="white", font=("Arial", 8), tags="targets")
            tsd_canvas.create_text(text_x, text_y+15, text=f"{dist:.1f}km {bearing:.0f}°", fill="white", font=("Arial", 8), tags="targets")
            if locked and missile_path and missile_path[4] == "outgoing":
                mx, my = missile_path[2], missile_path[3]
                mx += (x - mx) // 10
                my += (y - my) // 10
                missile_path[2], missile_path[3] = mx, my
                tsd_canvas.create_line(missile_path[0], missile_path[1], mx, my, fill="red", dash=(4, 2), tags="missile_path")
                if math.sqrt((mx - x)**2 + (my - y)**2) < 20:
                    missile_path = None

        if hit_marker:
            hx, hy, stage = hit_marker
            if stage > 0:
                size = 10 + stage * 10
                tsd_canvas.create_oval(hx-size, hy-size, hx+size, hy+size, outline="orange", tags="hit_marker")
                tsd_canvas.create_text(hx, hy, text="HIT", fill="red", font=("Arial", 10), tags="hit_marker")
                hit_marker[2] -= 1
            else:
                hit_marker = None

        jamming_status = "Jamming: ON" if jamming_active else "Jamming: OFF"
        tsd_canvas.create_text(270, 50, text=jamming_status, fill="white", font=("Arial", 10), tags="tsd_status")
        awacs_count = len([t for t in targets if t[5] == "Air" and t[6]]) + 1
        tsd_canvas.create_text(270, 80, text=f"AWACS: {awacs_count}", fill="white", font=("Arial", 10), tags="tsd_status")
        tsd_canvas.create_text(270, 110, text=f"Range: {radar_range} km", fill="white", font=("Arial", 10), tags="tsd_status")
        threat_level = sum(t[7] for t in targets if t[3] == "Enemy")
        tsd_canvas.create_text(270, 140, text=f"Threat: {threat_level}", fill="red" if threat_level > 10 else "white", font=("Arial", 10), tags="tsd_status")
    elif tsd_mode == "EW":
        suppressed = sum(1 for t in targets if t[3] == "Enemy" and random.random() < ecm_effect / 100)
        for i, target in enumerate(targets):
            x, y, _, iff, _, target_type, _, _ = target
            dist = math.sqrt((x - center_x)**2 + (y - center_y)**2) / 10
            if dist <= radar_range:
                angle = math.radians(i * 60)
                rx, ry = center_x + 90 * math.cos(angle), center_y + 90 * math.sin(angle)
                color = "red" if iff == "Enemy" else "green"
                tsd_canvas.create_oval(rx-5, ry-5, rx+5, ry+5, fill=color, tags="tsd_status")
                tsd_canvas.create_text(rx, ry-15, text=f"{iff} {target_type}", fill="white", font=("Arial", 8), tags="tsd_status")
                tsd_canvas.create_text(rx, ry+15, text=f"{dist:.1f}km", fill="white", font=("Arial", 8), tags="tsd_status")
        tsd_canvas.create_text(270, 50, text=f"Flares: {flares}", fill="white", font=("Arial", 10), tags="tsd_status")
        tsd_canvas.create_text(270, 80, text=f"Chaff: {chaff}", fill="white", font=("Arial", 10), tags="tsd_status")
        tsd_canvas.create_text(270, 110, text=f"Decoys: {decoys}", fill="white", font=("Arial", 10), tags="tsd_status")
        tsd_canvas.create_text(270, 140, text=f"ECM: {ecm_effect}%", fill="white", font=("Arial", 10), tags="tsd_status")
        tsd_canvas.create_text(270, 170, text=f"Suppressed: {suppressed}", fill="white", font=("Arial", 10), tags="tsd_status")
    root.after(500, update_radar)

# Tactical Buttons (3x3 Grid + Top Row)
def switch_radar_mode():
    global radar_mode, targets, target_locked, missile_path, hit_marker
    radar_mode = "Air-to-Ground" if radar_mode == "Air-to-Air" else "Air-to-Air"
    targets = [t for t in targets if t[5] == "SAM"]
    target_locked = None
    missile_path = None
    hit_marker = None
    tsd_canvas.delete("radar_mode")
    tsd_canvas.create_text(270, 450, text=f"Radar Mode: {radar_mode}", fill="white", font=("Arial", 10), tags="tsd_status")  # Below radar

def adjust_range():
    global radar_range
    radar_range = 200 if radar_range == 100 else 100
    tsd_canvas.delete("range_status")
    tsd_canvas.create_text(270, 470, text=f"Range Set: {radar_range} km", fill="yellow", font=("Arial", 10), tags="tsd_status")

def lock_target():
    global target_locked, targets, missile_path
    if targets:
        if target_locked is None:
            enemy_targets = [t for t in targets if t[3] == "Enemy"]
            if enemy_targets:
                target_locked = max(enemy_targets, key=lambda t: t[7])[2]
                for target in targets:
                    if target[2] == target_locked:
                        target[4] = True
                        if radar_mode == "Air-to-Ground":
                            missile_path = [center_x, center_y, target[0], target[1], "outgoing"]
        else:
            for target in targets:
                if target[2] == target_locked:
                    target[4] = False
            target_locked = None
            missile_path = None
    tsd_canvas.delete("lock_status")
    status = f"Locked: ID {target_locked}" if target_locked else "No Lock"
    tsd_canvas.create_text(270, 490, text=status, fill="yellow", font=("Arial", 10), tags="tsd_status")

def deploy_flares():
    global flares, incoming_missile
    if flares > 0:
        flares -= 1
        if incoming_missile and random.random() > 0.5:
            incoming_missile = None
        tsd_canvas.delete("flare_status")
        tsd_canvas.create_text(270, 510, text="Flares Deployed!", fill="yellow", font=("Arial", 10), tags="tsd_status")

def deploy_chaff():
    global chaff, incoming_missile
    if chaff > 0:
        chaff -= 1
        if incoming_missile and random.random() > 0.7:
            incoming_missile = None
        tsd_canvas.delete("chaff_status")
        tsd_canvas.create_text(270, 530, text="Chaff Deployed!", fill="yellow", font=("Arial", 10), tags="tsd_status")

def deploy_decoy():
    global decoys, incoming_missile
    if decoys > 0:
        decoys -= 1
        if incoming_missile and random.random() > 0.8:
            incoming_missile = None
        tsd_canvas.delete("decoy_status")
        tsd_canvas.create_text(270, 550, text="Decoy Deployed!", fill="yellow", font=("Arial", 10), tags="tsd_status")

def toggle_jamming():
    global jamming_active
    jamming_active = not jamming_active
    tsd_canvas.delete("jamming_toggle")
    status = "Jamming ON" if jamming_active else "Jamming OFF"
    tsd_canvas.create_text(270, 570, text=status, fill="yellow", font=("Arial", 10), tags="tsd_status")

def switch_tsd_mode():
    global tsd_mode
    tsd_mode = "EW" if tsd_mode == "RADAR" else "RADAR"
    tsd_canvas.delete("tsd_mode")
    tsd_canvas.create_text(270, 590, text=f"Mode: {tsd_mode}", fill="yellow", font=("Arial", 10), tags="tsd_status")

tsd_button_frame = tk.Frame(tsd_frame, bg="gray")
tsd_button_frame.pack(side=tk.BOTTOM, pady=5)
# 3x3 grid for 9 slots (8 buttons, 1 empty)
tk.Button(tsd_button_frame, text="RADAR MODE", command=switch_radar_mode, **button_style).grid(row=0, column=0, padx=5, pady=5)
tk.Button(tsd_button_frame, text="RANGE", command=adjust_range, **button_style).grid(row=0, column=1, padx=5, pady=5)
tk.Button(tsd_button_frame, text="LOCK", command=lock_target, **button_style).grid(row=0, column=2, padx=5, pady=5)
tk.Button(tsd_button_frame, text="FLARES", command=deploy_flares, **button_style).grid(row=1, column=0, padx=5, pady=5)
tk.Button(tsd_button_frame, text="CHAFF", command=deploy_chaff, **button_style).grid(row=1, column=1, padx=5, pady=5)
tk.Button(tsd_button_frame, text="DECOY", command=deploy_decoy, **button_style).grid(row=1, column=2, padx=5, pady=5)
tk.Button(tsd_button_frame, text="JAMMING", command=toggle_jamming, **button_style).grid(row=2, column=0, padx=5, pady=5)
tk.Button(tsd_button_frame, text="MODE", command=switch_tsd_mode, **button_style).grid(row=2, column=1, padx=5, pady=5)
# No button at (2, 2) as Tactical has 8 buttons

# ---------------------------
# Weapon Systems Display
# ---------------------------
wsd_label = tk.Label(wsd_frame, text="Weapon Systems Display", font=("Arial", 14))
wsd_label.pack(pady=5)
wsd_canvas = tk.Canvas(wsd_frame, bg="gray20", width=540, height=620)
wsd_canvas.pack(pady=5)

wsd_mode = "WEAPONS"
radio_commands = [
    "Alpha, engage bandit at 270°",
    "Bravo, cover my six",
    "Charlie, return to base",
    "AWACS, request target data",
    "All units, weapons free"
]
radio_index = 0
def update_weapon_status():
    global missile_count, bomb_count, target_locked, hit_marker, selected_loadout, guidance_mode
    wsd_canvas.delete("wsd_status", "fire_status", "ir_view")
    if wsd_mode == "WEAPONS":
        lock_status = f"Locked: ID {target_locked}" if target_locked else "No Lock"
        sequence = "Firing" if target_locked and random.random() > 0.7 else "Idle"
        waypoint_rel = f"Rel: {random.randint(-10, 10)}°" if target_locked else "N/A"
        if sequence == "Firing" and target_locked:
            hit_marker = [next(t for t in targets if t[2] == target_locked)[0], next(t for t in targets if t[2] == target_locked)[1], 3] if targets else None
        weapon_range = 150 if selected_loadout == "AA-12 Adder" else 50
        wsd_canvas.create_text(140, 50, text=f"Missiles: {missile_count}", fill="white", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 80, text=f"Bombs: {bomb_count}", fill="white", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 110, text=f"Gun: {'Ready' if random.random() > 0.1 else 'Jammed'}", fill="white", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 140, text=lock_status, fill="yellow", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 170, text=f"Sequence: {sequence}", fill="white", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 200, text=f"Range: {weapon_range} km", fill="white", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 230, text=f"Guidance: {guidance_mode}", fill="white", font=("Arial", 12), tags="wsd_status")
        wsd_canvas.create_text(140, 260, text=f"Waypoint: {waypoint_rel}", fill="white", font=("Arial", 12), tags="wsd_status")

        if target_locked:
            wsd_canvas.create_rectangle(400, 50, 500, 150, outline="white", fill="gray", tags="ir_view")
            wsd_canvas.create_oval(440, 90, 460, 110, fill="red" if guidance_mode == "IR" else "gray", tags="ir_view")
            wsd_canvas.create_text(450, 130, text=f"IR Lock: ID {target_locked}", fill="white", font=("Arial", 8), tags="ir_view")
    elif wsd_mode == "COMM":
        dl_friend = "; ".join([f"ID{t[2]}" for t in awacs_data["friendly"]])
        dl_enemy = "; ".join([f"ID{t[2]}" for t in awacs_data["enemy"]])
        wsd_canvas.create_text(270, 50, text=f"Radio: {radio_commands[radio_index]}", fill="white", font=("Arial", 10), tags="wsd_status")
        wsd_canvas.create_text(270, 80, text=f"Friendly DL: {dl_friend or 'None'}", fill="white", font=("Arial", 10), tags="wsd_status")
        wsd_canvas.create_text(270, 110, text=f"Enemy DL: {dl_enemy or 'None'}", fill="white", font=("Arial", 10), tags="wsd_status")
    root.after(3000, update_weapon_status)

# Weapon Buttons (3x3 Grid)
def select_weapon():
    global selected_loadout
    selected_loadout = "AA-12 Adder" if selected_loadout == "KAB-500 Bomb" else "KAB-500 Bomb"
    wsd_canvas.delete("loadout_status")
    wsd_canvas.create_text(140, 290, text=f"Selected: {selected_loadout}", fill="yellow", font=("Arial", 12), tags="wsd_status")

def fire_weapon():
    global missile_count, bomb_count, target_locked, targets, hit_marker, missile_path, selected_loadout, guidance_mode
    if target_locked and targets:
        target = next(t for t in targets if t[2] == target_locked)
        if selected_loadout == "AA-12 Adder" and missile_count > 0:
            missile_count -= 1
            wsd_canvas.delete("fire_status")
            wsd_canvas.create_text(140, 320, text="Missile Fired!", fill="red", font=("Arial", 12), tags="wsd_status")
            missile_path = [center_x, center_y, target[0], target[1], "outgoing"]
            hit_chance = 0.3 if guidance_mode == "Radar" else 0.5
            if random.random() > hit_chance:
                hit_marker = [target[0], target[1], 3]
                targets.remove(target)
                target_locked = None
        elif selected_loadout == "KAB-500 Bomb" and bomb_count > 0 and radar_mode == "Air-to-Ground":
            bomb_count -= 1
            wsd_canvas.delete("fire_status")
            wsd_canvas.create_text(140, 320, text="Bomb Dropped!", fill="red", font=("Arial", 12), tags="wsd_status")
            missile_path = [center_x, center_y, target[0], target[1], "outgoing"]
            if random.random() > 0.5:
                hit_marker = [target[0], target[1], 3]
                targets.remove(target)
                target_locked = None

def self_destruct():
    global missile_path
    if missile_path and missile_path[4] == "outgoing":
        hit_marker = [missile_path[2], missile_path[3], 3]
        missile_path = None
        wsd_canvas.delete("destruct_status")
        wsd_canvas.create_text(140, 350, text="Missile Self-Destructed!", fill="red", font=("Arial", 12), tags="wsd_status")

def switch_guidance():
    global guidance_mode
    guidance_mode = "IR" if guidance_mode == "Radar" else "Radar"
    wsd_canvas.delete("guidance_status")
    wsd_canvas.create_text(140, 380, text=f"Guidance: {guidance_mode}", fill="yellow", font=("Arial", 12), tags="wsd_status")

def switch_radio():
    global radio_index
    radio_index = (radio_index + 1) % len(radio_commands)
    wsd_canvas.delete("radio_status")
    wsd_canvas.create_text(140, 410, text=f"Sent: {radio_commands[radio_index]}", fill="yellow", font=("Arial", 10), tags="wsd_status")

def switch_wsd_mode():
    global wsd_mode
    wsd_mode = "COMM" if wsd_mode == "WEAPONS" else "WEAPONS"
    wsd_canvas.delete("wsd_mode")
    wsd_canvas.create_text(140, 440, text=f"Mode: {wsd_mode}", fill="yellow", font=("Arial", 12), tags="wsd_status")

wsd_button_frame = tk.Frame(wsd_frame, bg="gray")
wsd_button_frame.pack(side=tk.BOTTOM, pady=5)
tk.Button(wsd_button_frame, text="WEAPON SEL", command=select_weapon, **button_style).grid(row=0, column=0, padx=5, pady=5)
tk.Button(wsd_button_frame, text="FIRE", command=fire_weapon, **button_style).grid(row=0, column=1, padx=5, pady=5)
tk.Button(wsd_button_frame, text="SELF DESTR", command=self_destruct, **button_style).grid(row=0, column=2, padx=5, pady=5)
tk.Button(wsd_button_frame, text="GUIDANCE", command=switch_guidance, **button_style).grid(row=1, column=0, padx=5, pady=5)
tk.Button(wsd_button_frame, text="RADIO", command=switch_radio, **button_style).grid(row=1, column=1, padx=5, pady=5)
tk.Button(wsd_button_frame, text="MODE", command=switch_wsd_mode, **button_style).grid(row=1, column=2, padx=5, pady=5)

# Initial calls to update functions
update_nav_data()
update_radar()
update_weapon_status()

# Start the main event loop
root.mainloop()
