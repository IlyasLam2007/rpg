import pygame
import sys
import random
import pickle

# Initialize Pygame
pygame.init()

# Screen setup
WIDTH, HEIGHT = 1024, 768
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Eldoria: Legends of the Ancients")

# Colors
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
GREEN = (0, 255, 0)
RED = (255, 0, 0)
GRAY = (50, 50, 50)
DARKGRAY = (30, 30, 30)

# Clock
clock = pygame.time.Clock()

# --- Classes ---

# Player class with skills, experience, inventory, equipment
class Player(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.image = pygame.Surface((40, 60))
        self.image.fill(GREEN)
        self.rect = self.image.get_rect()
        self.rect.center = (WIDTH // 2, HEIGHT // 2)
        self.speed = 4
        self.health = 150
        self.max_health = 150
        self.experience = 0
        self.level = 1
        self.skills = {"swordsmanship": 1, "archery": 1, "magic": 1}
        self.inventory = []
        self.equipment = {"weapon": None, "armor": None}
        self.gold = 50
        self.dialogue_state = 0

    def update(self, keys):
        if keys[pygame.K_w]:
            self.rect.y -= self.speed
        if keys[pygame.K_s]:
            self.rect.y += self.speed
        if keys[pygame.K_a]:
            self.rect.x -= self.speed
        if keys[pygame.K_d]:
            self.rect.x += self.speed
        self.rect.clamp_ip(screen.get_rect())

    def gain_xp(self, amount):
        self.experience += amount
        if self.experience >= 100 * self.level:
            self.level_up()

    def level_up(self):
        self.level += 1
        self.experience = 0
        for skill in self.skills:
            self.skills[skill] += 1
        print(f"Leveled up! Level {self.level}")

    def attack(self, enemy):
        damage = random.randint(15, 30) + self.skills.get("swordsmanship", 1) * 2
        enemy.health -= damage
        print(f"Player attacked! Damage: {damage}")

    def heal(self, amount):
        self.health = min(self.max_health, self.health + amount)

# Enemy class
class Enemy(pygame.sprite.Sprite):
    def __init__(self, x, y, name="Goblin"):
        super().__init__()
        self.image = pygame.Surface((40, 40))
        self.image.fill(RED)
        self.rect = self.image.get_rect()
        self.rect.topleft = (x, y)
        self.health = 50
        self.name = name

    def update(self, player):
        # Simple AI: follow the player
        if self.rect.x < player.rect.x:
            self.rect.x += 1
        elif self.rect.x > player.rect.x:
            self.rect.x -= 1
        if self.rect.y < player.rect.y:
            self.rect.y += 1
        elif self.rect.y > player.rect.y:
            self.rect.y -= 1

    def attack(self, player):
        damage = random.randint(5, 15)
        player.health -= damage
        print(f"{self.name} attacked! Damage: {damage}")

# Item class
class Item(pygame.sprite.Sprite):
    def __init__(self, x, y, name):
        super().__init__()
        self.image = pygame.Surface((20, 20))
        self.image.fill(WHITE)
        self.rect = self.image.get_rect()
        self.rect.topleft = (x, y)
        self.name = name

# Quest class
class Quest:
    def __init__(self, title, description, reward, completed=False):
        self.title = title
        self.description = description
        self.reward = reward
        self.completed = completed

# Dialogue Tree
class DialogueTree:
    def __init__(self):
        self.nodes = {
            0: ("Welcome to Eldoria!", 1),
            1: ("Stranger, seek the ancient ruins.", 2),
            2: ("Beware of the shadows.", -1)
        }
        self.current_node = 0

    def get_next(self):
        text, next_node = self.nodes[self.current_node]
        self.current_node = next_node
        return text

    def reset(self):
        self.current_node = 0

# Save & Load
def save_game(player, enemies, items, quests):
    state = {
        'player': {
            'x': player.rect.x, 'y': player.rect.y, 'health': player.health,
            'xp': player.experience, 'level': player.level, 'skills': player.skills,
            'inventory': player.inventory, 'gold': player.gold
        },
        'enemies': [{'x': e.rect.x, 'y': e.rect.y, 'health': e.health, 'name': e.name} for e in enemies],
        'items': [{'x': i.rect.x, 'y': i.rect.y, 'name': i.name} for i in items],
        'quests': [{'title': q.title, 'completed': q.completed} for q in quests]
    }
    with open('eldoria_save.pkl', 'wb') as f:
        pickle.dump(state, f)
    print("Game Saved!")

def load_game():
    with open('eldoria_save.pkl', 'rb') as f:
        state = pickle.load(f)
    # Restore player
    p = state['player']
    player.rect.x = p['x']
    player.rect.y = p['y']
    player.health = p['health']
    player.experience = p['xp']
    player.level = p['level']
    player.skills = p['skills']
    player.inventory = p['inventory']
    player.gold = p['gold']
    # Restore enemies
    enemies.empty()
    for e_data in state['enemies']:
        e = Enemy(e_data['x'], e_data['y'], e_data['name'])
        e.health = e_data['health']
        enemies.add(e)
    # Restore items
    items.empty()
    for i_data in state['items']:
        items.add(Item(i_data['x'], i_data['y'], i_data['name']))
    # Restore quests
    quests.clear()
    for q_data in state['quests']:
        quests.append(Quest(q_data['title'], "", 0, q_data['completed']))
    print("Game Loaded!")

# --- Map & Tiles ---
TILE_SIZE = 32
MAP_WIDTH = WIDTH // TILE_SIZE
MAP_HEIGHT = HEIGHT // TILE_SIZE

# Simple map layout (0 = empty, 1 = obstacle)
sample_map = [
    [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
    [0,1,1,1,0,0,0,1,1,1,0,0,0,1,1,1,0,0,0,1,1,1,0,0,0],
    [0,1,0,0,0,1,0,0,0,1,0,1,0,0,0,1,0,1,0,0,0,1,0,1,0],
    [0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0],
    [0,0,0,1,0,0,0,1,0,0,0,1,0,0,0,1,0,0,0,1,0,0,0,1,0],
    [0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,0],
    # More rows can be added for a bigger map
]

# Function to draw map
def draw_map():
    for y in range(MAP_HEIGHT):
        for x in range(MAP_WIDTH):
            rect = pygame.Rect(x*TILE_SIZE, y*TILE_SIZE, TILE_SIZE, TILE_SIZE)
            if y < len(sample_map) and x < len(sample_map[y]):
                if sample_map[y][x] == 1:
                    pygame.draw.rect(screen, DARKGRAY, rect)
                else:
                    pygame.draw.rect(screen, GRAY, rect)
            else:
                pygame.draw.rect(screen, GRAY, rect)

# --- Sound & Music ---
# Placeholder for sound effects and background music
# Load your sound files here, e.g.:
# pygame.mixer.music.load('background.mp3')
# pygame.mixer.music.play(-1)

# For now, just a placeholder
def play_music():
    pass  # Insert your music playback code here

def play_sound(effect_name):
    pass  # Insert sound effect code here

# --- Main Game Loop ---
def main():
    # Initialize game states
    player = Player()
    enemies = pygame.sprite.Group()
    items = pygame.sprite.Group()
    quests = []

    # Spawn enemies and items
    for _ in range(8):
        x = random.randint(0, WIDTH - 40)
        y = random.randint(0, HEIGHT - 40)
        enemies.add(Enemy(x, y, "Goblin"))

    for _ in range(5):
        x = random.randint(0, WIDTH - 20)
        y = random.randint(0, HEIGHT - 20)
        items.add(Item(x, y, "Health Potion"))

    dialogue = DialogueTree()
    show_dialogue_flag = False
    dialogue_text = ""

    # Play background music placeholder
    play_music()

    while True:
        clock.tick(60)
        keys = pygame.key.get_pressed()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_SPACE:
                    # Attack
                    for e in enemies:
                        if player.rect.colliderect(e.rect.inflate(10, 10)):
                            player.attack(e)
                            # Play attack sound
                            play_sound("attack")
                elif event.key == pygame.K_F5:
                    save_game(player, enemies, items, quests)
                elif event.key == pygame.K_F9:
                    load_game()
                elif event.key == pygame.K_RETURN and show_dialogue_flag:
                    # Next dialogue
                    dialogue_text = dialogue.get_next()
                    if dialogue_text == "-1":
                        show_dialogue_flag = False
                elif event.key == pygame.K_c:
                    # Start dialogue
                    dialogue.reset()
                    dialogue_text = dialogue.get_next()
                    show_dialogue_flag = True

        if not show_dialogue_flag:
            # Update player
            player.update(keys)

            # Update enemies
            for e in enemies:
                e.update(player)
                if player.rect.colliderect(e.rect.inflate(10, 10)):
                    e.attack(player)

            # Pick up items
            for item in items:
                if player.rect.colliderect(item.rect):
                    player.inventory.append(item.name)
                    items.remove(item)
                    # Play pickup sound
                    play_sound("pickup")

            # Check death
            if player.health <= 0:
                print("You have fallen in battle.")
                pygame.quit()
                sys.exit()

        # Draw everything
        screen.fill(BLACK)
        draw_map()

        # Draw sprites
        screen.blit(player.image, player.rect)
        for e in enemies:
            screen.blit(e.image, e.rect)
        for item in items:
            screen.blit(item.image, item.rect)

        # UI overlays
        font = pygame.font.SysFont(None, 24)
        hp_text = font.render(f"HP: {player.health}/{player.max_health}", True, WHITE)
        lvl_text = font.render(f"Level: {player.level} XP: {player.experience}/{100 * player.level}", True, WHITE)
        inv_text = font.render(f"Inventory: {player.inventory}", True, WHITE)
        gold_text = font.render(f"Gold: {player.gold}", True, WHITE)

        screen.blit(hp_text, (10, 10))
        screen.blit(lvl_text, (10, 40))
        screen.blit(inv_text, (10, 70))
        screen.blit(gold_text, (10, 100))

        # Dialogue overlay
        if show_dialogue_flag:
            font_dialogue = pygame.font.SysFont(None, 30)
            dialogue_box = pygame.Surface((WIDTH - 40, 100))
            dialogue_box.fill(WHITE)
            screen.blit(dialogue_box, (20, HEIGHT - 120))
            dialogue_render = font_dialogue.render(dialogue_text, True, BLACK)
            screen.blit(dialogue_render, (30, HEIGHT - 100))
            instr = font.render("Press ENTER to continue", True, BLACK)
            screen.blit(instr, (30, HEIGHT - 60))

        pygame.display.flip()

if __name__ == "__main__":
    main()
