"""
getting_over_it_style.py
A Getting Over It–style physics game in pure Python + pygame.

Controls:
 - Move mouse to aim the hammer.
 - Left mouse button click: attempt to anchor the hammer tip at the mouse position (must be near terrain).
 - Release left mouse button: release the anchor.
 - Space: respawn (reset) to starting position.
 - R: reset camera zoom/position.

This is an original implementation inspired by the game mechanics (swinging with a long tool).
No original assets or copyrighted code is used.
"""

import math
import random
import sys
import pygame

# ---------- Configuration ----------
WIDTH, HEIGHT = 1200, 720
FPS = 60

PLAYER_RADIUS = 20            # the pot/cup radius
PLAYER_MASS = 1.0
GRAVITY = 1400.0              # pixels / s^2
AIR_FRICTION = 0.999
GROUND_FRICTION = 0.8

HAMMER_LENGTH = 230           # length of the hammer from player center to tip
HAMMER_THICKNESS = 6
ANCHOR_MAX_DIST = 12         # how close mouse must be to terrain to anchor
ANCHOR_SPRING_K = 1800.0     # spring stiffness when anchored
ANCHOR_DAMPING = 60.0

CAMERA_SMOOTH = 8.0
ZOOM_DEFAULT = 1.0
ZOOM_MIN, ZOOM_MAX = 0.5, 1.5

# ---------- Helper utilities ----------
vec2 = pygame.math.Vector2

def clamp(x, lo, hi):
    return max(lo, min(hi, x))

def length(v):
    return math.hypot(v.x, v.y)

def distance_point_segment(pt, a, b):
    # returns distance from pt to segment ab and closest point on segment
    ap = pt - a
    ab = b - a
    ab_len2 = ab.x * ab.x + ab.y * ab.y
    if ab_len2 == 0:
        return length(ap), a
    t = (ap.x * ab.x + ap.y * ab.y) / ab_len2
    t_clamped = clamp(t, 0.0, 1.0)
    closest = a + ab * t_clamped
    return length(pt - closest), closest

# ---------- Level geometry ----------
def generate_example_level():
    # returns list of segments (a, b) where a and b are vec2
    # We'll construct a tall, bumpy mountain using random segments
    segs = []

    # start at bottom-left
    x = -600
    y = HEIGHT + 400
    # create an incline that goes up to the right with ledges
    while x < 3000:
        step_x = random.randint(60, 160)
        step_y = random.randint(-180, -30)  # overall upward trend
        nx = x + step_x
        ny = y + step_y
        # Add intermediate small bumps to make climbing interesting
        bumps = random.randint(0, 2)
        if bumps == 0:
            segs.append((vec2(x, y), vec2(nx, ny)))
        else:
            midx = x + step_x * 0.5
            midy = y + step_y * 0.5 + random.randint(-80, 80)
            segs.append((vec2(x, y), vec2(midx, midy)))
            segs.append((vec2(midx, midy), vec2(nx, ny)))
        x, y = nx, ny

    # add a few floating ledges and vertical walls for variation
    for i in range(30):
        cx = random.randint(-200, 2800)
        cy = random.randint(-1200, 300)
        w = random.randint(60, 240)
        segs.append((vec2(cx, cy), vec2(cx + w, cy)))
    # add a big overhang wall
    segs.append((vec2(1200, -300), vec2(1400, 100)))
    segs.append((vec2(1400, 100), vec2(1600, -200)))
    segs.append((vec2(1800, 50), vec2(2100, 50)))
    return segs

# ---------- Game classes ----------
class Camera:
    def __init__(self, width, height):
        self.pos = vec2(0, 0)     # camera center world coords
        self.width = width
        self.height = height
        self.zoom = ZOOM_DEFAULT

    def world_to_screen(self, world_pos):
        screen_center = vec2(self.width / 2, self.height / 2)
        return (world_pos - self.pos) * self.zoom + screen_center

    def screen_to_world(self, screen_pos):
        screen_center = vec2(self.width / 2, self.height / 2)
        return (screen_pos - screen_center) / self.zoom + self.pos

    def update(self, target_pos, dt):
        # smooth follow
        diff = target_pos - self.pos
        self.pos += diff * clamp(CAMERA_SMOOTH * dt, 0, 1)

class Player:
    def __init__(self, start_pos):
        self.pos = vec2(start_pos)
        self.prev_pos = vec2(start_pos)  # for verlet integration
        self.radius = PLAYER_RADIUS
        self.on_ground = False

    def apply_gravity(self, dt):
        # Verlet integration: new_pos = pos + (pos - prev_pos) + acceleration * dt^2
        vel = (self.pos - self.prev_pos)
        self.prev_pos = vec2(self.pos)
        self.pos += vel * AIR_FRICTION
        # acceleration
        self.pos.y += GRAVITY * dt * dt

    def integrate(self, dt):
        # collisions and eventual damping handled externally
        pass

# ---------- Main game ----------
def main():
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    pygame.display.set_caption("Hammer-Climb (Getting Over It–style) — demo")
    clock = pygame.time.Clock()
    font = pygame.font.SysFont("Arial", 18)

    # Generate level
    segments = generate_example_level()

    # Starting position near the left-bottom of level
    start_pos = vec2(-400, HEIGHT + 150)

    player = Player(start_pos)

    # hammer anchor state
    anchored = False
    anchor_point = vec2(0, 0)
    rope_length = HAMMER_LENGTH

    camera = Camera(WIDTH, HEIGHT)
    camera.pos = vec2(start_pos)

    running = True
    mouse_down = False
    show_debug = False

    # For simplistic collision resolution we'll store some bounce/repel adjustments
    def resolve_player_terrain_collisions():
        # Push player out of terrain segments if overlapping (player is circle)
        # We'll test all segments; this is O(n) and fine for small levels.
        for a, b in segments:
            dist, closest = distance_point_segment(player.pos, a, b)
            if dist < player.radius:
                # push player out along normal
                if dist == 0:
                    # degenerate case: push up
                    n = vec2(0, -1)
                else:
                    n = (player.pos - closest) / dist
                push = (player.radius - dist)
                player.pos += n * push
                # basic friction: damp previous position to remove energy
                # compute velocity and reduce its component along the normal
                v = player.pos - player.prev_pos
                vel_normal = v.dot(n)
                v = v - n * vel_normal * (1 + 0.2)  # some bounce factor
                player.prev_pos = player.pos - v * GROUND_FRICTION

    # function to check if a click near terrain segment (within ANCHOR_MAX_DIST)
    def find_anchor_candidate(world_mouse):
        best = None
        best_dist = 1e9
        for a, b in segments:
            dist, closest = distance_point_segment(world_mouse, a, b)
            if dist < best_dist:
                best_dist = dist
                best = (dist, closest, a, b)
        if best and best[0] <= ANCHOR_MAX_DIST:
            return best[1]  # closest point
        return None

    # small helper to draw dashed line
    def draw_dashed_line(surf, color, p1, p2, dash_len=8, width=1):
        total = (p2 - p1).length()
        if total == 0:
            return
        nseg = int(total / dash_len)
        if nseg == 0:
            pygame.draw.line(surf, color, p1, p2, width)
            return
        for i in range(nseg):
            t0 = i / nseg
            t1 = (i + 0.5) / nseg
            a = p1 + (p2 - p1) * t0
            b = p1 + (p2 - p1) * t1
            pygame.draw.line(surf, color, a, b, width)

    # Main loop
    while running:
        dt = clock.tick(FPS) / 1000.0
        for ev in pygame.event.get():
            if ev.type == pygame.QUIT:
                running = False
            elif ev.type == pygame.KEYDOWN:
                if ev.key == pygame.K_ESCAPE:
                    running = False
                elif ev.key == pygame.K_SPACE:
                    # Respawn
                    player.pos = vec2(start_pos)
                    player.prev_pos = vec2(start_pos)
                    anchored = False
                elif ev.key == pygame.K_r:
                    camera.zoom = ZOOM_DEFAULT
                    camera.pos = vec2(player.pos)
                elif ev.key == pygame.K_d:
                    show_debug = not show_debug
                elif ev.key == pygame.K_PLUS or ev.key == pygame.K_EQUALS:
                    camera.zoom = clamp(camera.zoom + 0.1, ZOOM_MIN, ZOOM_MAX)
                elif ev.key == pygame.K_MINUS:
                    camera.zoom = clamp(camera.zoom - 0.1, ZOOM_MIN, ZOOM_MAX)
            elif ev.type == pygame.MOUSEBUTTONDOWN and ev.button == 1:
                mouse_down = True
                world_mouse = camera.screen_to_world(vec2(ev.pos))
                candidate = find_anchor_candidate(world_mouse)
                if candidate is not None:
                    anchored = True
                    anchor_point = vec2(candidate)
                    rope_length = (player.pos - anchor_point).length()
                    if rope_length < 20:
                        rope_length = 20.0
                else:
                    # if not near terrain, allow anchor to anywhere if very close to hammer tip (for a more permissive feel)
                    # check hammer tip position
                    # hammer tip pos:
                    mouse_world = world_mouse
                    tip_dir = (mouse_world - player.pos)
                    if tip_dir.length() == 0:
                        tip_dir = vec2(1, 0)
                    tip_dir = tip_dir.normalize()
                    hammer_tip = player.pos + tip_dir * HAMMER_LENGTH
                    if (hammer_tip - mouse_world).length() < 20:
                        anchored = True
                        anchor_point = vec2(mouse_world)
                        rope_length = (player.pos - anchor_point).length()
            elif ev.type == pygame.MOUSEBUTTONUP and ev.button == 1:
                mouse_down = False
                anchored = False
            elif ev.type == pygame.MOUSEWHEEL:
                # zoom with wheel
                camera.zoom = clamp(camera.zoom + ev.y * 0.05, ZOOM_MIN, ZOOM_MAX)

        # Integrate player physics (Verlet)
        player.apply_gravity(dt)

        # If anchored: enforce distance constraint via spring/damping to simulate rope/hammer
        if anchored:
            # target distance = rope_length
            to_anchor = player.pos - anchor_point
            dist = to_anchor.length()
            if dist == 0:
                dirn = vec2(0, -1)
            else:
                dirn = to_anchor / dist
            # spring displacement
            disp = dist - rope_length
            # compute velocity (approx)
            vel = (player.pos - player.prev_pos) / dt if dt > 0 else vec2(0, 0)
            rel_vel = vel.dot(dirn)
            # spring impulse
            force_mag = -ANCHOR_SPRING_K * disp - ANCHOR_DAMPING * rel_vel
            # convert force to position correction (Verlet-friendly)
            # acceleration = F / m
            accel = dirn * (force_mag / PLAYER_MASS)
            # apply position correction (Verlet)
            player.pos += accel * dt * dt
            # small extra constraint correction to avoid drift
            # keep max stretch
            max_stretch = rope_length * 1.25
            if dist > max_stretch:
                player.pos = anchor_point + dirn * max_stretch

        # simple terrain collision resolution
        resolve_player_terrain_collisions()

        # camera target is player.pos, but also bias upwards when anchored to show more of level
        camera_target = vec2(player.pos)
        if anchored:
            # aim camera between player and anchor for better view
            camera_target = (player.pos * 0.65 + anchor_point * 0.35)
        camera.update(camera_target, dt)

        # rendering
        screen.fill((45, 55, 70))

        # draw background grid for reference
        # We'll draw subtle lines every 200 px world units
        grid_color = (30, 35, 45)
        world_top_left = camera.screen_to_world(vec2(0, 0))
        world_bottom_right = camera.screen_to_world(vec2(WIDTH, HEIGHT))
        gx0 = int(world_top_left.x // 200 * 200)
        gy0 = int(world_top_left.y // 200 * 200)
        for gx in range(gx0 - 400, int(world_bottom_right.x) + 400, 200):
            a = camera.world_to_screen(vec2(gx, world_top_left.y - 4000))
            b = camera.world_to_screen(vec2(gx, world_bottom_right.y + 4000))
            pygame.draw.line(screen, grid_color, a, b, 1)
        for gy in range(gy0 - 4000, int(world_bottom_right.y) + 4000, 200):
            a = camera.world_to_screen(vec2(world_top_left.x - 4000, gy))
            b = camera.world_to_screen(vec2(world_bottom_right.x + 4000, gy))
            pygame.draw.line(screen, grid_color, a, b, 1)

        # draw segments (terrain)
        for a, b in segments:
            sa = camera.world_to_screen(a)
            sb = camera.world_to_screen(b)
            pygame.draw.line(screen, (200, 200, 190), sa, sb, 4)

        # draw anchor if anchored
        if anchored:
            sa = camera.world_to_screen(anchor_point)
            pygame.draw.circle(screen, (255, 200, 60), (int(sa.x), int(sa.y)), max(3, int(6 * camera.zoom)))
            # draw dashed line for rope
            sp = camera.world_to_screen(player.pos)
            draw_dashed_line(screen, (220, 220, 255), sp, sa, dash_len=12, width=2)

        # draw hammer (from player to mouse direction)
        mouse_screen = vec2(pygame.mouse.get_pos())
        world_mouse = camera.screen_to_world(mouse_screen)
        aim_dir = world_mouse - player.pos
        if aim_dir.length() == 0:
            aim_dir = vec2(1, 0)
        aim_dir = aim_dir.normalize()
        hammer_tip = player.pos + aim_dir * HAMMER_LENGTH
        # hammer parts in screen coords
        sp_player = camera.world_to_screen(player.pos)
        sp_tip = camera.world_to_screen(hammer_tip)
        # thick line for hammer shaft
        hw = max(1, int(HAMMER_THICKNESS * camera.zoom))
        pygame.draw.line(screen, (180, 180, 190), sp_player, sp_tip, hw)
        # hammer head (small rectangle) at tip
        # compute perpendicular
        perp = vec2(-aim_dir.y, aim_dir.x) * (12)
        head_poly = [
            camera.world_to_screen(hammer_tip + perp * 0.6),
            camera.world_to_screen(hammer_tip - perp * 0.6),
            camera.world_to_screen(hammer_tip - aim_dir * 18 - perp * 0.6),
            camera.world_to_screen(hammer_tip - aim_dir * 18 + perp * 0.6),
        ]
        pygame.draw.polygon(screen, (100, 100, 110), head_poly)

        # draw player (pot) as circle with a little rim
        p_screen = camera.world_to_screen(player.pos)
        pygame.draw.circle(screen, (120, 140, 160), (int(p_screen.x), int(p_screen.y)), max(4, int(player.radius * camera.zoom)))
        pygame.draw.circle(screen, (40, 40, 50), (int(p_screen.x), int(p_screen.y)), max(4, int(player.radius * camera.zoom)), 2)

        # draw HUD
        info = [
            "Left click to anchor hammer to terrain (must be near a surface).",
            "Hold and swing to build momentum. Release to drop anchor.",
            "Space = respawn • Mouse wheel = zoom • +/- = zoom in/out • D = debug",
        ]
        for i, s in enumerate(info):
            surf = font.render(s, True, (230, 230, 230))
            screen.blit(surf, (10, 10 + i * 20))

        # Debug info
        if show_debug:
            dbg = [
                f"pos: {player.pos.x:.1f}, {player.pos.y:.1f}",
                f"vel_est: {(player.pos - player.prev_pos).length()/dt if dt>0 else 0:.1f} px/s",
                f"anchored: {anchored}",
                f"rope_len: {rope_length:.1f}",
                f"anchor: {anchor_point.x:.1f}, {anchor_point.y:.1f}"
            ]
            for i, s in enumerate(dbg):
                surf = font.render(s, True, (255, 200, 100))
                screen.blit(surf, (10, HEIGHT - 20 * (len(dbg) - i)))

        # small trophy: mark top-most y reached
        # (for fun, draw a little flag where the highest player y reached would be)
        # We can compute the highest by tracking global, but keep light: we skip for now.

        pygame.display.flip()

    pygame.quit()
    sys.exit()

if __name__ == "__main__":
    main()

