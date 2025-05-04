import pygame
import random
import sys

# Initialize Pygame
pygame.init()

# Screen settings
WIDTH, HEIGHT = 600, 400
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Jump Over the Barrels")

# Colors
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED   = (200, 0, 0)
BLUE  = (0, 100, 255)
YELLOW = (255, 255, 0)  # For hammer
GREEN = (0, 255, 0)  # For enemies
ORANGE = (255, 165, 0)  # For enemy fire

# Clock and font
clock = pygame.time.Clock()
FPS = 75
font = pygame.font.SysFont(None, 36)

# Game states
menu = True
game_over = False

# Player settings
player = pygame.Rect(100, HEIGHT - 60, 40, 40)
player_vel_y = 0
on_ground = True
player_speed = 5
facing_right = True  # Tracks which direction player is facing

# Throwing hammer settings
hammers = []
hammer_speed = 10
can_throw_hammer = True
hammer_cooldown = 500  # Time in milliseconds between throws
last_throw_time = pygame.time.get_ticks()

# Barrel settings (reduced speed)
barrels = []
barrel_speed = 4  # Slower barrel speed
SPAWN_BARREL = pygame.USEREVENT + 1
pygame.time.set_timer(SPAWN_BARREL, 1000)

# Vertical enemies settings
enemies = []
enemy_speed = 2  # Speed of enemies moving up and down
SPAWN_ENEMY = pygame.USEREVENT + 2
pygame.time.set_timer(SPAWN_ENEMY, 2000)

# Enemy fire settings
enemy_fireballs = []
enemy_fire_speed = 7  # Speed of enemy fireballs
FIRE_RATE = 1000  # Time in milliseconds between enemy shots
last_enemy_shot_time = pygame.time.get_ticks()

score = 0

def draw_text(text, size, color, center):
    font_obj = pygame.font.SysFont(None, size)
    surf = font_obj.render(text, True, color)
    rect = surf.get_rect(center=center)
    screen.blit(surf, rect)

def reset_game():
    global player, barrels, enemies, score, player_vel_y, on_ground, game_over, hammers, facing_right, enemy_fireballs
    player.x, player.y = 100, HEIGHT - 60
    player_vel_y = 0
    on_ground = True
    facing_right = True
    barrels.clear()
    enemies.clear()
    hammers.clear()
    enemy_fireballs.clear()
    score = 0
    game_over = False

# Game loop
while True:
    screen.fill(WHITE)
    keys = pygame.key.get_pressed()

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
        if event.type == SPAWN_BARREL and not game_over and not menu:
            barrels.append(pygame.Rect(random.randint(0, WIDTH-30), -40, 30, 30))
        if event.type == SPAWN_ENEMY and not game_over and not menu:
            enemy_height = random.randint(50, HEIGHT-150)  # Random position for vertical movement
            enemy_rect = pygame.Rect(random.randint(100, WIDTH-100), enemy_height, 40, 40)
            enemies.append(enemy_rect)

    if menu:
        draw_text("Jump Over the Barrels", 48, BLUE, (WIDTH//2, HEIGHT//3))
        draw_text("Press SPACE to Start", 32, BLACK, (WIDTH//2, HEIGHT//2))
        if keys[pygame.K_SPACE]:
            menu = False
            reset_game()

    elif not game_over:
        # Movement
        if keys[pygame.K_LEFT]:
            player.x -= player_speed
            facing_right = False
        if keys[pygame.K_RIGHT]:
            player.x += player_speed
            facing_right = True
        if keys[pygame.K_SPACE] and on_ground:
            player_vel_y = -16
            on_ground = False

        # Throw hammer
        if keys[pygame.K_f] and can_throw_hammer:
            if facing_right:
                hammers.append(pygame.Rect(player.x + player.width, player.y + player.height // 4, 20, 10))  # Right
            else:
                hammers.append(pygame.Rect(player.x - 20, player.y + player.height // 4, 20, 10))  # Left
            can_throw_hammer = False
            last_throw_time = pygame.time.get_ticks()

        # Cooldown for hammer
        if not can_throw_hammer and pygame.time.get_ticks() - last_throw_time > hammer_cooldown:
            can_throw_hammer = True

        # Gravity
        player_vel_y += 1
        player.y += player_vel_y
        if player.y >= HEIGHT - 60:
            player.y = HEIGHT - 60
            player_vel_y = 0
            on_ground = True

        # Keep player on screen
        player.x = max(0, min(WIDTH - player.width, player.x))

        # Move barrels
        for barrel in barrels[:]:
            barrel.y += barrel_speed
            pygame.draw.rect(screen, RED, barrel)

            if player.colliderect(barrel):
                game_over = True
            if barrel.y > HEIGHT:
                barrels.remove(barrel)
                score += 1

        # Move and draw enemies (vertical movement)
        for enemy in enemies[:]:
            if enemy.y <= 0 or enemy.y >= HEIGHT - 50:  # Reverse direction at top and bottom
                enemy_speed = -enemy_speed
            enemy.y += enemy_speed
            pygame.draw.rect(screen, GREEN, enemy)

            if player.colliderect(enemy):
                game_over = True

            # Enemy firing mechanism
            if pygame.time.get_ticks() - last_enemy_shot_time > FIRE_RATE:
                enemy_fireballs.append(pygame.Rect(enemy.x + enemy.width // 2, enemy.y + enemy.height, 10, 10))
                last_enemy_shot_time = pygame.time.get_ticks()

        # Move and draw fireballs from enemies
        for fireball in enemy_fireballs[:]:
            fireball.x -= enemy_fire_speed  # Fireball moves left towards the player
            pygame.draw.rect(screen, ORANGE, fireball)

            if fireball.colliderect(player):
                game_over = True
            if fireball.x < 0:  # Remove fireball when it goes off screen
                enemy_fireballs.remove(fireball)

        # Move and draw hammers
        for hammer in hammers[:]:
            if facing_right:
                hammer.x += hammer_speed
            else:
                hammer.x -= hammer_speed
            pygame.draw.rect(screen, YELLOW, hammer)

            # Remove hammer when it goes off-screen or hits a barrel
            for barrel in barrels[:]:
                if hammer.colliderect(barrel):
                    barrels.remove(barrel)
                    hammers.remove(hammer)
                    score += 1
                    break

            # Check if hammer hits enemy
            for enemy in enemies[:]:
                if hammer.colliderect(enemy):
                    enemies.remove(enemy)
                    hammers.remove(hammer)
                    score += 5  # Reward more points for killing an enemy
                    break

            if hammer.x > WIDTH or hammer.x < 0:
                hammers.remove(hammer)

        # Draw player
        pygame.draw.rect(screen, BLUE, player)

        # Display score
        draw_text(f"Score: {score}", 28, BLACK, (70, 30))

    else:
        draw_text("Game Over!", 48, RED, (WIDTH//2, HEIGHT//3))
        draw_text("Press R to Restart", 32, BLACK, (WIDTH//2, HEIGHT//2))
        if keys[pygame.K_r]:
            reset_game()

    pygame.display.flip()
    clock.tick(FPS)
