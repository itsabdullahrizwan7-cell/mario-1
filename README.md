import pygame
import sys
import math

pygame.init()
WIDTH, HEIGHT = 1000, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("1-2 Player Mario Platformer")
clock = pygame.time.Clock()
FPS = 60

# Colors
SKY = (135, 206, 235)
GREEN = (60, 180, 75)
BROWN = (139, 69, 19)
RED = (220, 50, 50)
BLUE = (60, 120, 220)
GOLD = (255, 215, 0)
BLACK = (20, 20, 20)
WHITE = (255, 255, 255)
BRICK = (200, 100, 50)
DARK_GREEN = (0, 150, 0)
GRAY = (100, 100, 100, 150)

# --- Platform + Input Setup ---
PLATFORM = None # 'windows' or 'android'
player_count = 1
game_started = False

# Touch button rects for Android
touch_buttons = {
    'p1_left': pygame.Rect(20, HEIGHT - 100, 80, 80),
    'p1_right': pygame.Rect(120, HEIGHT - 100, 80, 80),
    'p1_jump': pygame.Rect(WIDTH - 100, HEIGHT - 100, 80, 80),
    'p2_left': pygame.Rect(20, HEIGHT - 200, 80, 80),
    'p2_right': pygame.Rect(120, HEIGHT - 200, 80, 80),
    'p2_jump': pygame.Rect(WIDTH - 100, HEIGHT - 200, 80, 80),
}

def get_touch_input():
    """Returns dict of pressed actions for touch controls"""
    pressed = {'p1_left': False, 'p1_right': False, 'p1_jump': False,
               'p2_left': False, 'p2_right': False, 'p2_jump': False}
    if PLATFORM!= 'android':
        return pressed

    for finger in range(10): # check up to 10 touch points
        try:
            pos = pygame.mouse.get_pos() if finger == 0 else None
            if pygame.mouse.get_pressed()[0] and pos: # mouse as touch for testing
                for btn, rect in touch_buttons.items():
                    if rect.collidepoint(pos):
                        pressed[btn] = True
        except:
            pass
    return pressed

class Player:
    def __init__(self, x, y, controls, color, name, player_num):
        self.spawn_x = x
        self.spawn_y = y
        self.controls = controls # keyboard keys
        self.color = color
        self.name = name
        self.player_num = player_num # 1 or 2
        self.reset()

    def reset(self):
        self.rect = pygame.Rect(self.spawn_x, self.spawn_y, 40, 50)
        self.vel_x = 0
        self.vel_y = 0
        self.speed = 5
        self.jump_power = -16
        self.on_ground = False
        self.coins = 0
        self.big = False
        self.invincible = 0
        self.facing = 1
        self.alive = True
        self.finished = False

    def update(self, keys, touch_input, platforms, bricks):
        if not self.alive or self.finished:
            return

        self.vel_x = 0

        # Check input based on platform
        left = keys[self.controls['left']] if PLATFORM == 'windows' else touch_input[f'p{self.player_num}_left']
        right = keys[self.controls['right']] if PLATFORM == 'windows' else touch_input[f'p{self.player_num}_right']
        jump = keys[self.controls['jump']] if PLATFORM == 'windows' else touch_input[f'p{self.player_num}_jump']

        if left:
            self.vel_x = -self.speed
            self.facing = -1
        if right:
            self.vel_x = self.speed
            self.facing = 1
        if jump and self.on_ground:
            self.vel_y = self.jump_power
            self.on_ground = False

        self.vel_y += 0.8
        if self.vel_y > 15:
            self.vel_y = 15

        self.rect.x += self.vel_x
        for obj in platforms + bricks:
            if obj.alive and self.rect.colliderect(obj.rect):
                if self.vel_x > 0:
                    self.rect.right = obj.rect.left
                if self.vel_x < 0:
                    self.rect.left = obj.rect.right

        self.rect.y += self.vel_y
        self.on_ground = False
        for obj in platforms + bricks:
            if obj.alive and self.rect.colliderect(obj.rect):
                if self.vel_y > 0:
                    self.rect.bottom = obj.rect.top
                    self.vel_y = 0
                    self.on_ground = True
                elif self.vel_y < 0:
                    self.rect.top = obj.rect.bottom
                    self.vel_y = 0
                    if isinstance(obj, Brick):
                        obj.hit(self.big)

        if self.invincible > 0:
            self.invincible -= 1

    def get_hit(self):
        if self.invincible > 0:
            return False
        if self.big:
            self.big = False
            self.rect = pygame.Rect(self.rect.x, self.rect.y + 20, 40, 50)
            self.invincible = 90
            return False
        self.alive = False
        return True

    def draw(self, screen, camera_x):
        if not self.alive:
            return
        draw_rect = self.rect.copy()
        draw_rect.x -= camera_x
        if self.invincible > 0 and self.invincible % 10 < 5:
            return
        if self.big:
            draw_rect.height = 70
            draw_rect.y -= 20
        pygame.draw.rect(screen, self.color, draw_rect)
        face_y = draw_rect.y + 20
        pygame.draw.rect(screen, (255, 220, 177), (draw_rect.x, face_y, draw_rect.width, 15))
        eye_x = draw_rect.centerx + 5 * self.facing
        pygame.draw.circle(screen, BLACK, (eye_x, face_y + 7), 3)

class Goomba:
    def __init__(self, x, y):
        self.rect = pygame.Rect(x, y, 40, 40)
        self.vel_x = -2
        self.alive = True

    def update(self, platforms, bricks):
        if not self.alive:
            return
        self.rect.x += self.vel_x
        on_edge = True
        feet_rect = pygame.Rect(self.rect.x + 5, self.rect.bottom, self.rect.width - 10, 5)
        for obj in platforms + bricks:
            if obj.alive and feet_rect.colliderect(obj.rect):
                on_edge = False
            if obj.alive and self.rect.colliderect(obj.rect) and obj.rect.y < self.rect.bottom - 10:
                self.vel_x *= -1
                self.rect.x += self.vel_x * 2
        if on_edge:
            self.vel_x *= -1

    def draw(self, screen, camera_x):
        if not self.alive:
            return
        draw_rect = self.rect.copy()
        draw_rect.x -= camera_x
        pygame.draw.ellipse(screen, BROWN, draw_rect)

class Koopa:
    def __init__(self, x, y):
        self.rect = pygame.Rect(x, y, 40, 50)
        self.vel_x = -1.5
        self.state = 'walk'
        self.alive = True

    def update(self, platforms, bricks):
        if not self.alive:
            return
        if self.state == 'walk' or self.state == 'sliding':
            self.rect.x += self.vel_x
            on_edge = True
            feet_rect = pygame.Rect(self.rect.x + 5, self.rect.bottom, self.rect.width - 10, 5)
            for obj in platforms + bricks:
                if obj.alive and feet_rect.colliderect(obj.rect):
                    on_edge = False
                if obj.alive and self.rect.colliderect(obj.rect) and obj.rect.y < self.rect.bottom - 10:
                    self.vel_x *= -1
            if on_edge and self.state == 'walk':
                self.vel_x *= -1

    def get_stomped(self):
        if self.state == 'walk':
            self.state = 'shell'
            self.vel_x = 0
            self.rect = pygame.Rect(self.rect.x, self.rect.y + 20, 40, 30)
        elif self.state == 'shell':
            self.state = 'sliding'
            self.vel_x = 8
        elif self.state == 'sliding':
            self.vel_x = 0
            self.state = 'shell'

    def draw(self, screen, camera_x):
        if not self.alive:
            return
        draw_rect = self.rect.copy()
        draw_rect.x -= camera_x
        if self.state == 'walk':
            pygame.draw.ellipse(screen, DARK_GREEN, draw_rect)
            pygame.draw.circle(screen, GOLD, (draw_rect.centerx, draw_rect.y + 10), 10)
        else:
            pygame.draw.ellipse(screen, DARK_GREEN, draw_rect)

class Brick:
    def __init__(self, x, y):
        self.rect = pygame.Rect(x, y, 40, 40)
        self.alive = True

    def hit(self, player_is_big):
        if player_is_big:
            self.alive = False

    def draw(self, screen, camera_x):
        if not self.alive:
            return
        draw_rect = self.rect.copy()
        draw_rect.x -= camera_x
        pygame.draw.rect(screen, BRICK, draw_rect)
        pygame.draw.rect(screen, BLACK, draw_rect, 2)

class Mushroom:
    def __init__(self, x, y):
        self.rect = pygame.Rect(x, y, 30, 30)
        self.collected = False

    def draw(self, screen, camera_x):
        if self.collected:
            return
        draw_rect = self.rect.copy()
        draw_rect.x -= camera_x
        pygame.draw.ellipse(screen, RED, (draw_rect.x, draw_rect.y, 30, 20))
        pygame.draw.rect(screen, WHITE, (draw_rect.x + 8, draw_rect.y + 15, 14, 15))

class Flagpole:
    def __init__(self, x, y):
        self.rect = pygame.Rect(x, y, 10, 300)
        self.flag_rect = pygame.Rect(x + 10, y + 20, 40, 25)

    def draw(self, screen, camera_x):
        pole = self.rect.copy()
        pole.x -= camera_x
        flag = self.flag_rect.copy()
        flag.x -= camera_x
        pygame.draw.rect(screen, WHITE, pole)
        pygame.draw.rect(screen, GREEN, flag)

class Platform:
    def __init__(self, x, y, w, h):
        self.rect = pygame.Rect(x, y, w, h)
        self.alive = True
    def draw(self, screen, camera_x):
        draw_rect = self.rect.copy()
        draw_rect.x -= camera_x
        color = GREEN if self.rect.y == 550 else BROWN
        pygame.draw.rect(screen, color, draw_rect)

LEVELS = [
    {
        'platforms': [(0, 550, 2500, 50), (300, 450, 200, 20), (600, 350, 200, 20), (900, 450, 150, 20), (1200, 400, 200, 20)],
        'bricks': [(400, 350, 40, 40), (440, 350, 40, 40), (1000, 350, 40, 40)],
        'coins': [(350, 410), (650, 310), (940, 410), (1250, 360)],
        'goombas': [(700, 510), (1300, 360)],
        'koopas': [(1000, 500)],
        'mushrooms': [(620, 320)],
        'flag_x': 2000
    },
    {
        'platforms': [(0, 550, 3000, 50), (400, 450, 100, 20), (700, 350, 100, 20), (1000, 250, 200, 20), (1400, 400, 300, 20)],
        'bricks': [(500, 350, 40, 40), (800, 250, 40, 40), (840, 250, 40, 40), (1500, 300, 40, 40)],
        'coins': [(450, 410), (750, 310), (1050, 210), (1550, 260)],
        'goombas': [(900, 510), (1600, 510)],
        'koopas': [(1200, 500), (1800, 500)],
        'mushrooms': [(820, 220)],
        'flag_x': 2800
    }
]

def load_level(level_num, players):
    data = LEVELS[level_num]
    for p in players:
        p.reset()
    platforms = [Platform(x, y, w, h) for x, y, w, h in data['platforms']]
    bricks = [Brick(x, y) for x, y, w, h in data['bricks']]
    coins = [pygame.Rect(x, y, 20, 20) for x, y in data['coins']]
    enemies = [Goomba(x, y) for x, y in data['goombas']]
    enemies += [Koopa(x, y) for x, y in data['koopas']]
    mushrooms = [Mushroom(x, y) for x, y in data['mushrooms']]
    flagpole = Flagpole(data['flag_x'], 250)
    return platforms, bricks, coins, enemies, mushrooms, flagpole

def draw_text(text, size, x, y, color=BLACK):
    font = pygame.font.Font(None, size)
    img = font.render(text, True, color)
    screen.blit(img, (x, y))

def draw_button(rect, text, color):
    s = pygame.Surface((rect.width, rect.height), pygame.SRCALPHA)
    s.fill(color)
    screen.blit(s, rect.topleft)
    draw_text(text, 40, rect.centerx - len(text)*8, rect.centery - 15, WHITE)

# --- Setup Players ---
p1_controls = {'left': pygame.K_a, 'right': pygame.K_d, 'jump': pygame.K_w}
p2_controls = {'left': pygame.K_LEFT, 'right': pygame.K_RIGHT, 'jump': pygame.K_UP}

player1 = Player(100, 400, p1_controls, RED, "P1", 1)
player2 = Player(150, 400, p2_controls, BLUE, "P2", 2)
players = [player1, player2]

current_level = 0
platforms, bricks, coins, enemies, mushrooms, flagpole = load_level(current_level, players[:player_count])
camera_x = 0
level_complete = False

while True:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
        if event.type == pygame.KEYDOWN:
            if not game_started and PLATFORM:
                if event.key == pygame.K_1:
                    player_count = 1
                    game_started = True
                    platforms, bricks, coins, enemies, mushrooms, flagpole = load_level(0, players[:player_count])
                if event.key == pygame.K_2:
                    player_count = 2
                    game_started = True
                    platforms, bricks, coins, enemies, mushrooms, flagpole = load_level(0, players[:player_count])
        if event.type == pygame.MOUSEBUTTONDOWN and not PLATFORM:
            # Platform select buttons
            win_btn = pygame.Rect(WIDTH//2 - 200, 300, 400, 80)
            and_btn = pygame.Rect(WIDTH//2 - 200, 400, 400, 80)
            if win_btn.collidepoint(event.pos):
                PLATFORM = 'windows'
            if and_btn.collidepoint(event.pos):
                PLATFORM = 'android'

    keys = pygame.key.get_pressed()
    touch_input = get_touch_input()

    # --- Platform Select Screen ---
    if not PLATFORM:
        screen.fill(SKY)
        draw_text("SELECT PLATFORM", 80, WIDTH//2 - 280, 150)
        draw_button(pygame.Rect(WIDTH//2 - 200, 300, 400, 80), "WINDOWS", BLUE)
        draw_button(pygame.Rect(WIDTH//2 - 200, 400, 400, 80), "ANDROID", GREEN)
        pygame.display.flip()
        clock.tick(FPS)
        continue

    # --- Player Count Select ---
    if not game_started:
        screen.fill(SKY)
        draw_text("SELECT PLAYERS", 80, WIDTH//2 - 280, 150)
        draw_text("Press 1 for 1 Player", 50, WIDTH//2 - 200, 300, RED)
        draw_text("Press 2 for 2 Players", 50, WIDTH//2 - 200, 370, BLUE)
        if PLATFORM == 'windows':
            draw_text("P1: WASD + W to jump", 30, WIDTH//2 - 150, 450)
            draw_text("P2: Arrows + Up to jump", 30, WIDTH//2 - 150, 480)
        else:
            draw_text("P1: Bottom left/right + right jump", 30, WIDTH//2 - 200, 450)
            draw_text("P2: Mid left/right + right jump", 30, WIDTH//2 - 200, 480)
        pygame.display.flip()
        clock.tick(FPS)
        continue

    active_players = players[:player_count]

    if not level_complete:
        for player in active_players:
            player.update(keys, touch_input, platforms, bricks)

        for enemy in enemies:
            if enemy.alive:
                enemy.update(platforms, bricks)
                for player in active_players:
                    if player.alive and not player.finished and player.rect.colliderect(enemy.rect):
                        if player.vel_y > 0 and player.rect.bottom < enemy.rect.centery + 10:
                            if isinstance(enemy, Koopa):
                                enemy.get_stomped()
                            else:
                                enemy.alive = False
                            player.vel_y = -10
                        else:
                            if isinstance(enemy, Koopa) and enemy.state == 'shell':
                                enemy.get_stomped()
                            else:
                                player.get_hit()

                if isinstance(enemy, Koopa) and enemy.state == 'sliding':
                    for other in enemies:
                        if other!= enemy and other.alive and enemy.rect.colliderect(other.rect):
                            other.alive = False

        for coin in coins[:]:
            for player in active_players:
                if player.alive and player.rect.colliderect(coin):
                    coins.remove(coin)
                    player.coins += 1
                    break

        for shroom in mushrooms:
            if not shroom.collected:
                for player in active_players:
                    if player.alive and player.rect.colliderect(shroom.rect):
                        shroom.collected = True
                        if not player.big:
                            player.big = True
                            player.rect = pygame.Rect(player.rect.x, player.rect.y - 20, 40, 70)
                        break

        for player in active_players:
            if player.alive and not player.finished:
                if player.rect.colliderect(flagpole.rect) or player.rect.colliderect(flagpole.flag_rect):
                    player.finished = True

        for player in active_players:
            if player.alive and player.rect.y > HEIGHT:
                player.get_hit()

        alive_players = [p for p in active_players if p.alive]
        if all(p.finished for p in alive_players) or not alive_players:
            level_complete = True

        if alive_players:
            avg_x = sum(p.rect.centerx for p in alive_players) / len(alive_players)
            camera_x = avg_x - WIDTH // 2
            if camera_x < 0:
                camera_x = 0

    # --- Draw ---
    screen.fill(SKY)

    for plat in platforms:
        plat.draw(screen, camera_x)
    for brick in bricks:
        brick.draw(screen, camera_x)

    for coin in coins:
        draw_rect = coin.copy()
        draw_rect.x -= camera_x
        pygame.draw.ellipse(screen, GOLD, draw_rect)

    for shroom in mushrooms:
        shroom.draw(screen, camera_x)
    for enemy in enemies:
        enemy.draw(screen, camera_x)
    flagpole.draw(screen, camera_x)

    for player in active_players:
        player.draw(screen, camera_x)

    # Draw touch buttons on Android
    if PLATFORM == 'android':
        for name, rect in touch_buttons.items():
            if player_count == 1 and name.startswith('p2'):
                continue
            draw_button(rect, name.split('_')[1][0].upper(), GRAY)

    # UI
    draw_text(f"Players: {player_count} | {PLATFORM.upper()}", 24, 20, 20)
    draw_text(f"Level: {current_level + 1}", 24, 20, 45)
    for i, player in enumerate(active_players):
        status = "Win" if player.finished else "Out" if not player.alive else f"Coins: {player.coins}"
        draw_text(f"{player.name}: {status}", 24, 20, 70 + i * 25, player.color)

    if level_complete:
        if current_level + 1 < len(LEVELS):
            draw_text("LEVEL COMPLETE! Press N for next", 50, WIDTH//2 - 300, HEIGHT//2 - 40)
            if keys[pygame.K_n]:
                current_level += 1
                platforms, bricks, coins, enemies, mushrooms, flagpole = load_level(current_level, players[:player_count])
                level_complete = False
        else:
            draw_text("YOU BEAT ALL LEVELS!", 60, WIDTH//2 - 280, HEIGHT//2 - 40)

        draw_text("Press R to restart", 30, WIDTH//2 - 120, HEIGHT//2 + 20)
        if keys[pygame.K_r]:
            current_level = 0
            platforms, bricks, coins, enemies, mushrooms, flagpole = load_level(current_level, players[:player_count])
            level_complete = False

    pygame.display.flip()
    clock.tick(FPS)
