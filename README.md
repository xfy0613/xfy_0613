import pygame
import sys
import random
import math
import time

pygame.init()

# 視窗
WIDTH, HEIGHT = 800, 500
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Pong 旋球版 - AI可旋球")

WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
ARROW_COLOR = (255, 0, 0)

clock = pygame.time.Clock()

# 球拍
paddle_width, paddle_height = 10, 100
left_paddle_x, left_paddle_y = 30, HEIGHT//2 - paddle_height//2
right_paddle_x, right_paddle_y = WIDTH-40, HEIGHT//2 - paddle_height//2
base_paddle_speed = 5
ai_base_speed = 3

# 球
ball_radius = 12

def reset_ball():
    global ball_x, ball_y, ball_speed_x, ball_speed_y, spin_x, spin_y, float_effect, can_spin, ai_can_spin
    ball_x, ball_y = WIDTH//2, HEIGHT//2
    angle = random.uniform(-math.pi/4, math.pi/4)
    direction = random.choice([-1,1])
    speed = 5
    ball_speed_x = direction * speed * math.cos(angle)
    ball_speed_y = speed * math.sin(angle)
    spin_x = 0
    spin_y = 0
    float_effect = False
    can_spin = False
    ai_can_spin = False

reset_ball()
ball_max_speed = 25

# 分數
left_score = 0
right_score = 0
font = pygame.font.SysFont(None, 48)

# 技能設定
skill_duration = 2
spin_x = 0
spin_y = 0
spin_end_time = 0
float_effect = False
can_spin = False
ai_can_spin = False

def ball_rect():
    return pygame.Rect(ball_x-ball_radius, ball_y-ball_radius, ball_radius*2, ball_radius*2)

def paddle_rect(x, y):
    return pygame.Rect(x, y, paddle_width, paddle_height)

def reflect_ball(paddle_y, is_ai=False):
    global can_spin, ai_can_spin
    paddle_center = paddle_y + paddle_height/2
    relative_intersect = (ball_y - paddle_center) / (paddle_height/2)
    max_bounce_angle = math.radians(75)
    bounce_angle = relative_intersect * max_bounce_angle
    speed = min(math.hypot(ball_speed_x, ball_speed_y)*1.05, ball_max_speed)
    direction = 1 if ball_speed_x < 0 else -1
    new_speed_x = direction * speed * math.cos(bounce_angle)
    new_speed_y = speed * math.sin(bounce_angle)
    if is_ai:
        ai_can_spin = True
    else:
        can_spin = True
    return new_speed_x, new_speed_y

running = True
while running:
    now = time.time()
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

        if event.type == pygame.KEYDOWN:
            if can_spin:
                speed = math.hypot(ball_speed_x, ball_speed_y)
                # 立即使用旋球技能
                if event.key == pygame.K_a:  # 左旋
                    ball_speed_x = -abs(speed)
                    spin_x = -0.5*speed
                    spin_y = 0
                    float_effect = False
                    spin_end_time = now + skill_duration
                    can_spin = False
                elif event.key == pygame.K_d:  # 右旋
                    ball_speed_x = abs(speed)
                    spin_x = 0.5*speed
                    spin_y = 0
                    float_effect = False
                    spin_end_time = now + skill_duration
                    can_spin = False
                elif event.key == pygame.K_q:  # 上旋
                    ball_speed_y = -abs(speed)
                    spin_x = 0
                    spin_y = -0.5*speed
                    float_effect = False
                    spin_end_time = now + skill_duration
                    can_spin = False
                elif event.key == pygame.K_z:  # 下旋
                    ball_speed_y = abs(speed)
                    spin_x = 0
                    spin_y = 0.5*speed
                    float_effect = False
                    spin_end_time = now + skill_duration
                    can_spin = False
                elif event.key == pygame.K_SPACE:  # 漂浮球
                    spin_x = 0
                    spin_y = 0
                    float_effect = True
                    spin_end_time = now + skill_duration
                    can_spin = False

    keys = pygame.key.get_pressed()
    # 玩家控制
    if keys[pygame.K_w] and left_paddle_y>0:
        left_paddle_y -= base_paddle_speed
    if keys[pygame.K_s] and left_paddle_y<HEIGHT-paddle_height:
        left_paddle_y += base_paddle_speed

    # AI控制
    ai_speed = min(ai_base_speed + right_score*0.5, 8)
    if ball_speed_x>0:
        time_to_reach = (right_paddle_x - ball_x)/ball_speed_x
        predicted_y = ball_y + ball_speed_y*time_to_reach
        while predicted_y<0 or predicted_y>HEIGHT:
            if predicted_y<0: predicted_y = -predicted_y
            if predicted_y>HEIGHT: predicted_y = 2*HEIGHT - predicted_y
        target_y = predicted_y + random.uniform(-30,30)
    else:
        target_y = HEIGHT/2

    if right_paddle_y + paddle_height/2 < target_y:
        right_paddle_y += ai_speed
    elif right_paddle_y + paddle_height/2 > target_y:
        right_paddle_y -= ai_speed
    right_paddle_y = max(0, min(HEIGHT-paddle_height, right_paddle_y))

    # AI技能碰球後立即使用
    if ai_can_spin:
        speed = math.hypot(ball_speed_x, ball_speed_y)
        choice = random.choice(['a','d','q','z','space'])
        if choice=='a': ball_speed_x=-abs(speed); spin_x=-0.5*speed; spin_y=0; float_effect=False
        elif choice=='d': ball_speed_x=abs(speed); spin_x=0.5*speed; spin_y=0; float_effect=False
        elif choice=='q': ball_speed_y=-abs(speed); spin_x=0; spin_y=-0.5*speed; float_effect=False
        elif choice=='z': ball_speed_y=abs(speed); spin_x=0; spin_y=0.5*speed; float_effect=False
        else: spin_x=0; spin_y=0; float_effect=True
        spin_end_time = now + skill_duration
        ai_can_spin = False

    if now>spin_end_time:
        spin_x=0
        spin_y=0
        float_effect=False

    # 球移動
    if float_effect:
        ball_y += math.sin(time.time()*5)*10
        if ball_y - ball_radius <0:
            ball_y=ball_radius
        elif ball_y + ball_radius>HEIGHT:
            ball_y=HEIGHT-ball_radius

    ball_x += ball_speed_x + spin_x
    ball_y += ball_speed_y + spin_y

    # 球碰上下邊界
    if ball_y-ball_radius<=0 or ball_y+ball_radius>=HEIGHT:
        ball_speed_y*=-1
        spin_y*=-1

    ball_r = ball_rect()
    left_r = paddle_rect(left_paddle_x, left_paddle_y)
    right_r = paddle_rect(right_paddle_x, right_paddle_y)

    # 球拍碰撞
    if ball_r.colliderect(left_r) and ball_speed_x<0:
        ball_speed_x, ball_speed_y = reflect_ball(left_paddle_y)
    if ball_r.colliderect(right_r) and ball_speed_x>0:
        ball_speed_x, ball_speed_y = reflect_ball(right_paddle_y, is_ai=True)

    # 計分
    if ball_x<0:
        right_score += 1
        reset_ball()
    elif ball_x>WIDTH:
        left_score += 1
        reset_ball()

    # 畫面
    screen.fill(BLACK)
    pygame.draw.circle(screen, WHITE, (int(ball_x), int(ball_y)), ball_radius)
    pygame.draw.rect(screen, WHITE, left_r)
    pygame.draw.rect(screen, WHITE, right_r)

    # 中線
    for y in range(0, HEIGHT, 30):
        pygame.draw.line(screen, WHITE, (WIDTH//2, y), (WIDTH//2, y+15), 3)

    # 分數
    left_text = font.render(str(left_score), True, WHITE)
    right_text = font.render(str(right_score), True, WHITE)
    screen.blit(left_text, (WIDTH//4,20))
    screen.blit(right_text, (WIDTH*3//4,20))

    # 箭頭提示旋球方向
    if spin_x!=0 or spin_y!=0 or float_effect:
        end_x = max(ball_radius, min(WIDTH-ball_radius, ball_x + spin_x*10))
        end_y = max(ball_radius, min(HEIGHT-ball_radius, ball_y + spin_y*10))
        pygame.draw.line(screen, ARROW_COLOR, (ball_x, ball_y), (end_x, end_y), 3)

    pygame.display.flip()
    clock.tick(60)

pygame.quit()
sys.exit()
