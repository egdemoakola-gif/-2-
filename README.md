import pymem
import pymem.process
import math
import time
import threading
import tkinter as tk
from tkinter import ttk

# ---------- Оффсеты CS2 (май 2026) ----------
dwLocalPlayerController = 0x19E24E8
dwEntityList = 0x1A42B68
dwViewAngles = 0x1A58A30
dwForceAttack = 0x1798

m_hPlayerPawn = 0x7BC
m_iHealth = 0x324
m_vOldOrigin = 0x13AC
m_iTeamNum = 0x3C3
m_vecViewOffset = 0x6F0
m_iObserverMode = 0x1E4
m_hObserverTarget = 0x1E8

# ---------- Глобальное состояние ----------
pm = None
client_dll = None
game_ready = False

# ---------- Настройки (все управляются через панель) ----------
class Config:
    aimbot_enabled = False
    trigger_enabled = False
    spinbot_enabled = False
    thirdperson_enabled = False
    
    aim_fov = 120.0
    aim_smooth = 0.15
    aim_hard = True
    aim_bone = 0
    
    trigger_delay_ms = 0
    
    spin_speed = 360.0
    spin_axis = "yaw"      # yaw / pitch / both
    spin_direction = "clockwise"

config = Config()

# ---------- Spinbot состояние ----------
class SpinbotState:
    last_time = 0.0
    zigzag_dir = 1

spin_state = SpinbotState()

# ---------- Функции работы с памятью ----------
def init_mem():
    global pm, client_dll, game_ready
    try:
        pm = pymem.Pymem("cs2.exe")
        client_dll = pymem.process.module_from_name(pm.process_handle, "client.dll").lpBaseOfDll
        game_ready = True
        print("[SWILL] CS2 attached")
        return True
    except:
        game_ready = False
        return False

def read_int(addr):
    return pm.read_int(addr)

def read_float(addr):
    return pm.read_float(addr)

def read_vec3(addr):
    return pm.read_vec3(addr)

def write_float(addr, val):
    pm.write_float(addr, val)

def write_vec3(addr, vec):
    pm.write_vec3(addr, vec)

def get_local_player():
    ctrl = read_int(client_dll + dwLocalPlayerController)
    if not ctrl: return 0
    pawn_ptr = read_int(ctrl + m_hPlayerPawn) & 0xFFFF
    if not pawn_ptr: return 0
    entry = read_int(client_dll + dwEntityList + 0x8 * (pawn_ptr >> 9))
    if not entry: return 0
    return read_int(entry + 0x78 * (pawn_ptr & 0x1FF))

def get_entity_pawn(idx):
    entry = read_int(client_dll + dwEntityList + 0x8 * (idx >> 9))
    if not entry: return 0
    ctrl = read_int(entry + 0x78 * (idx & 0x1FF))
    if not ctrl: return 0
    pawn_ptr = read_int(ctrl + m_hPlayerPawn) & 0xFFFF
    if not pawn_ptr: return 0
    ent_entry = read_int(client_dll + dwEntityList + 0x8 * (pawn_ptr >> 9))
    if not ent_entry: return 0
    return read_int(ent_entry + 0x78 * (pawn_ptr & 0x1FF))

def get_team(pawn):
    return read_int(pawn + m_iTeamNum)

def get_health(pawn):
    return read_int(pawn + m_iHealth)

def get_pos(pawn):
    return read_vec3(pawn + m_vOldOrigin)

def get_view_offset(pawn):
    return read_vec3(pawn + m_vecViewOffset)

def get_bone_pos(pawn, bone):
    origin = get_pos(pawn)
    view = get_view_offset(pawn)
    if bone == 0:  # голова
        return (origin[0] + view[0], origin[1] + view[1], origin[2] + view[2] + 4.0)
    elif bone == 1:
        return (origin[0] + view[0], origin[1] + view[1], origin[2] + view[2] - 2.0)
    else:
        return (origin[0] + view[0], origin[1] + view[1], origin[2] + view[2] - 12.0)

def calc_angle(local_eye, target):
    dx = target[0] - local_eye[0]
    dy = target[1] - local_eye[1]
    dz = target[2] - local_eye[2]
    yaw = math.degrees(math.atan2(dy, dx))
    pitch = math.degrees(math.atan2(-dz, math.hypot(dx, dy)))
    return [pitch, yaw]

def set_third_person(enable):
    pawn = get_local_player()
    if not pawn: return
    if enable:
        pm.write_int(pawn + m_iObserverMode, 1)
        pm.write_int(pawn + m_hObserverTarget, pawn & 0xFFFF)
    else:
        pm.write_int(pawn + m_iObserverMode, 0)
        pm.write_int(pawn + m_hObserverTarget, 0)

# ---------- Поиск цели для аима ----------
def get_best_target(local_pawn, local_eye, local_team, fov):
    best_dist = fov + 1
    best_pos = None
    for i in range(1, 65):
        ent = get_entity_pawn(i)
        if not ent or ent == local_pawn: continue
        if get_health(ent) <= 0: continue
        if get_team(ent) == local_team: continue
        tpos = get_bone_pos(ent, config.aim_bone)
        ang = calc_angle(local_eye, tpos)
        cur = read_vec3(client_dll + dwViewAngles)
        delta_pitch = ang[0] - cur[0]
        delta_yaw = ang[1] - cur[1]
        if delta_pitch > 180: delta_pitch -= 360
        if delta_yaw > 180: delta_yaw -= 360
        dist = math.hypot(delta_pitch, delta_yaw)
        if dist < best_dist:
            best_dist = dist
            best_pos = tpos
    return best_pos, best_dist

# ---------- Основной поток читов ----------
def cheat_loop():
    global game_ready
    last_spin_time = time.time()
    
    while True:
        if not game_ready:
            if init_mem():
                game_ready = True
            else:
                time.sleep(0.5)
                continue
        
        # Проверка что игра ещё жива
        try:
            pm.read_int(client_dll + dwLocalPlayerController)
        except:
            game_ready = False
            time.sleep(0.5)
            continue
        
        local_pawn = get_local_player()
        if not local_pawn:
            time.sleep(0.01)
            continue
        
        local_team = get_team(local_pawn)
        local_pos = get_pos(local_pawn)
        view_off = get_view_offset(local_pawn)
        local_eye = (local_pos[0] + view_off[0], local_pos[1] + view_off[1], local_pos[2] + view_off[2])
        current_angles = read_vec3(client_dll + dwViewAngles)
        cur_pitch, cur_yaw = current_angles[0], current_angles[1]
        
        # ---- Автовыстрел (по любому врагу, даже сзади) ----
        if config.trigger_enabled:
            for i in range(1, 65):
                ent = get_entity_pawn(i)
                if not ent or ent == local_pawn: continue
                if get_health(ent) <= 0: continue
                if get_team(ent) == local_team: continue
                pm.write_int(client_dll + dwForceAttack, 5)
                time.sleep(0.01)
                pm.write_int(client_dll + dwForceAttack, 4)
                time.sleep(max(0.005, config.trigger_delay_ms / 1000.0))
                break
        
        # ---- Aimbot ----
        target_acquired = False
        if config.aimbot_enabled:
            best_pos, dist = get_best_target(local_pawn, local_eye, local_team, config.aim_fov)
            if best_pos and dist <= config.aim_fov:
                target_angles = calc_angle(local_eye, best_pos)
                if config.aim_hard:
                    write_vec3(client_dll + dwViewAngles, target_angles)
                else:
                    new_pitch = cur_pitch + (target_angles[0] - cur_pitch) * config.aim_smooth
                    new_yaw = cur_yaw + (target_angles[1] - cur_yaw) * config.aim_smooth
                    write_vec3(client_dll + dwViewAngles, [new_pitch, new_yaw])
                target_acquired = True
        
        # ---- Spinbot (только если цель не захвачена и спины вкл) ----
        if config.spinbot_enabled and not target_acquired:
            now = time.time()
            delta = min(now - last_spin_time, 0.033)
            last_spin_time = now
            step = config.spin_speed * delta
            
            new_yaw = cur_yaw
            new_pitch = cur_pitch
            
            if config.spin_axis == "yaw":
                if config.spin_direction == "clockwise":
                    new_yaw += step
                elif config.spin_direction == "counter":
                    new_yaw -= step
                elif config.spin_direction == "zigzag":
                    new_yaw += step * spin_state.zigzag_dir
                    if abs(new_yaw - cur_yaw) > 180:
                        spin_state.zigzag_dir *= -1
            elif config.spin_axis == "pitch":
                if config.spin_direction == "clockwise":
                    new_pitch += step
                else:
                    new_pitch -= step
                if new_pitch > 89: new_pitch = 89
                if new_pitch < -89: new_pitch = -89
            elif config.spin_axis == "both":
                new_yaw += step
                new_pitch += step * 0.5
                if new_pitch > 89: new_pitch = 89
                if new_pitch < -89: new_pitch = -89
            
            if new_yaw > 180: new_yaw -= 360
            if new_yaw < -180: new_yaw += 360
            write_vec3(client_dll + dwViewAngles, [new_pitch, new_yaw])
        
        # ---- Третье лицо ----
        if config.thirdperson_enabled and not target_acquired:
            set_third_person(True)
        elif not config.thirdperson_enabled:
            set_third_person(False)
        
        time.sleep(0.005)

# ---------- GUI панель (всё управление здесь) ----------
class ControlPanel:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("SWILL CS2 External Panel")
        self.root.geometry("420x600")
        self.root.attributes("-topmost", True)
        
        self.status_label = tk.Label(self.root, text="🔴 Ожидание CS2...", fg="red", font=("Arial", 10))
        self.status_label.pack(pady=5)
        
        notebook = ttk.Notebook(self.root)
        notebook.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Вкладка Aimbot / Trigger
        aim_frame = ttk.Frame(notebook)
        notebook.add(aim_frame, text="Aimbot & Trigger")
        self.build_aim_tab(aim_frame)
        
        # Вкладка Spinbot
        spin_frame = ttk.Frame(notebook)
        notebook.add(spin_frame, text="Spinbot")
        self.build_spin_tab(spin_frame)
        
        # Вкладка Third Person
        third_frame = ttk.Frame(notebook)
        notebook.add(third_frame, text="Third Person")
        self.build_third_tab(third_frame)
        
        self.update_status_loop()
        
    def build_aim_tab(self, parent):
        # Aimbot ON/OFF
        self.aim_var = tk.BooleanVar(value=config.aimbot_enabled)
        tk.Checkbutton(parent, text="Включить Aimbot", variable=self.aim_var, command=self.toggle_aim).pack(anchor=tk.W, pady=2)
        
        # Hard/Smooth
        self.hard_var = tk.BooleanVar(value=config.aim_hard)
        tk.Checkbutton(parent, text="Hard Aim (мгновенный)", variable=self.hard_var, command=self.toggle_hard).pack(anchor=tk.W)
        
        # FOV слайдер
        tk.Label(parent, text="FOV аима (0-360)").pack(anchor=tk.W)
        self.fov_slider = ttk.Scale(parent, from_=0, to=360, orient=tk.HORIZONTAL, value=config.aim_fov, command=self.set_fov)
        self.fov_slider.pack(fill=tk.X)
        
        # Smooth слайдер
        tk.Label(parent, text="Smooth (0.01=жёсткий, 1.0=плавный)").pack(anchor=tk.W)
        self.smooth_slider = ttk.Scale(parent, from_=0.01, to=1.0, orient=tk.HORIZONTAL, value=config.aim_smooth, command=self.set_smooth)
        self.smooth_slider.pack(fill=tk.X)
        
        # Bone
        tk.Label(parent, text="Кость для аима: 0=голова 1=шея 2=грудь").pack(anchor=tk.W)
        self.bone_entry = tk.Entry(parent)
        self.bone_entry.insert(0, str(config.aim_bone))
        self.bone_entry.pack(fill=tk.X)
        tk.Button(parent, text="Применить кость", command=self.set_bone).pack(pady=2)
        
        # Trigger
        self.trig_var = tk.BooleanVar(value=config.trigger_enabled)
        tk.Checkbutton(parent, text="Включить автовыстрел (по любому врагу)", variable=self.trig_var, command=self.toggle_trig).pack(anchor=tk.W, pady=5)
        
        tk.Label(parent, text="Задержка автовыстрела (мс)").pack(anchor=tk.W)
        self.delay_slider = ttk.Scale(parent, from_=0, to=200, orient=tk.HORIZONTAL, value=config.trigger_delay_ms, command=self.set_delay)
        self.delay_slider.pack(fill=tk.X)
    
    def build_spin_tab(self, parent):
        self.spin_var = tk.BooleanVar(value=config.spinbot_enabled)
        tk.Checkbutton(parent, text="Включить Spinbot (крутилку)", variable=self.spin_var, command=self.toggle_spin).pack(anchor=tk.W, pady=5)
        
        tk.Label(parent, text="Скорость вращения (град/сек)").pack(anchor=tk.W)
        self.spin_speed = ttk.Scale(parent, from_=0, to=720, orient=tk.HORIZONTAL, value=config.spin_speed, command=self.set_spin_speed)
        self.spin_speed.pack(fill=tk.X)
        
        tk.Label(parent, text="Ось вращения").pack(anchor=tk.W)
        self.axis_combo = ttk.Combobox(parent, values=["yaw", "pitch", "both"], state="readonly")
        self.axis_combo.set(config.spin_axis)
        self.axis_combo.pack(fill=tk.X)
        self.axis_combo.bind("<<ComboboxSelected>>", self.set_axis)
        
        tk.Label(parent, text="Направление").pack(anchor=tk.W)
        self.dir_combo = ttk.Combobox(parent, values=["clockwise", "counter", "zigzag"], state="readonly")
        self.dir_combo.set(config.spin_direction)
        self.dir_combo.pack(fill=tk.X)
        self.dir_combo.bind("<<ComboboxSelected>>", self.set_dir)
    
    def build_third_tab(self, parent):
        self.third_var = tk.BooleanVar(value=config.thirdperson_enabled)
        tk.Checkbutton(parent, text="Включить Third Person (третье лицо)", variable=self.third_var, command=self.toggle_third).pack(anchor=tk.W, pady=10)
        tk.Label(parent, text="Вид от третьего лица без sv_cheats", fg="gray").pack()
    
    def toggle_aim(self): config.aimbot_enabled = self.aim_var.get()
    def toggle_hard(self): config.aim_hard = self.hard_var.get()
    def toggle_trig(self): config.trigger_enabled = self.trig_var.get()
    def toggle_spin(self): config.spinbot_enabled = self.spin_var.get()
    def toggle_third(self): config.thirdperson_enabled = self.third_var.get()
    
    def set_fov(self, v): config.aim_fov = float(v)
    def set_smooth(self, v): config.aim_smooth = float(v)
    def set_delay(self, v): config.trigger_delay_ms = float(v)
    def set_spin_speed(self, v): config.spin_speed = float(v)
    def set_axis(self, e): config.spin_axis = self.axis_combo.get()
    def set_dir(self, e): config.spin_direction = self.dir_combo.get()
    def set_bone(self):
        try: config.aim_bone = int(self.bone_entry.get())
        except: pass
    
    def update_status_loop(self):
        if game_ready:
            self.status_label.config(text="✅ CS2 подключён | Читы активны", fg="green")
        else:
            self.status_label.config(text="🔴 Ожидание CS2... (запусти игру)", fg="red")
        self.root.after(500, self.update_status_loop)
    
    def run(self):
        self.root.mainloop()

# ---------- Запуск ----------
if __name__ == "__main__":
    # Поток читов
    t = threading.Thread(target=cheat_loop, daemon=True)
    t.start()
    # Панель
    panel = ControlPanel()
    panel.run()
