title: Python语法学习项目
author: 钱晓豪
tags:
  - Python
categories:
  - Python
  - 项目源码
date: 2020-03-16 17:27:00
---
学习完python后，网上找的程序源码，当做语法学习完成后的总结，包括 `Python基本语法` 、 `模块` 、`面向对象`
# 项目
## 功能

1、一个玩家控制的飞机并发射子弹、若干台敌机以及滚动的背景
2、玩家控制飞机左右移动，子弹自动发射  
3、子弹碰到敌机子弹和敌机消失，玩家控制的飞机碰到敌机游戏自动退出

## 类中的静态方法
* 情况：没使用类或对象的属性或方法
* 方法：@staticmethod
* 调用：类名.方法
* 理解:单例模式


## 项目源码
**plane_main.py** 源码：
```
import pygame
from plane_sprites import *
#主窗体大小
SCREEN_RECT = pygame.Rect(0,0,480,700)
#主窗体刷新速度
FRAME_PER_SEC = 60
#创建事件
CREATE_ENEMY_EVENT = pygame.USEREVENT
HERO_FIRE_EVENT = pygame.USEREVENT + 1

class PlaneGame(object):
	  #初始化函数
    def __init__(self):
    	  #创建主窗体
        self.screen = pygame.display.set_mode(SCREEN_RECT.size)
        #创建定时器
        self.clock = pygame.time.Clock()
        #创建对象
        self.__create_sprites()
        #定时器定时发送事件消息
        pygame.time.set_timer(CREATE_ENEMY_EVENT, 1000)
        pygame.time.set_timer(HERO_FIRE_EVENT, 500)

    def __create_sprites(self):
    	  #精灵组类操作，相同类型的对象放在一起，只调用一个函数，减小代码量
        #背景对象，两个背景是为了实现无缝滚动，一个背景会出现空白背景
        bg1 = Background()
        bg2 = Background(True)
        self.back_group = pygame.sprite.Group(bg1,bg2)
        #敌机对象，若干个，在之后添加
        self.enemy_group = pygame.sprite.Group()
        #玩家控制的飞机对象
        self.hero = Hero()
        self.hero_group = pygame.sprite.Group(self.hero)

    def start_game(self):
    	  #主循环，保证主窗体可以一直运行
        while True:
        	  #设置时长，保证while巡检1s执行FRAME_PER_SEC次
            self.clock.tick(FRAME_PER_SEC)
            #监听主窗体消息
            self.__event_handle()
            #碰撞检测，调用接口，几个对象坐标重合时的处理
            self.__check_collide()
            #设置对象坐标位置
            self.__update_sprites()
            #重绘主窗体
            pygame.display.update()

    def __event_handle(self):
        for event in pygame.event.get():
        	  #点击窗体关闭按钮
            if event.type == pygame.QUIT:
                PlaneGame.__game_over()
            #定时器定时发送
            elif event.type == CREATE_ENEMY_EVENT:
                enemy = Enemy()
                self.enemy_group.add(enemy)
            elif event.type == HERO_FIRE_EVENT:
                self.hero.fire()
        #每个消息判断是不是左右按键，控制飞机左右移动
        keys_pressed = pygame.key.get_pressed()
        if keys_pressed[pygame.K_RIGHT]:
            self.hero.speed = 1
        elif keys_pressed[pygame.K_LEFT]:
            self.hero.speed = -1
        else:
            self.hero.speed = 0

    def __check_collide(self):
   		  #调用接口
        pygame.sprite.groupcollide(self.hero.bullets, self.enemy_group, True, True)
        enemies = pygame.sprite.spritecollide(self.hero, self.enemy_group, True)
        if len(enemies) > 0:
            self.hero.kill()
            PlaneGame.__game_over()

    def __update_sprites(self):
        self.back_group.update()
        self.back_group.draw(self.screen)
        self.enemy_group.update()
        self.enemy_group.draw(self.screen)
        self.hero_group.update()
        self.hero_group.draw(self.screen)
        self.hero.bullets.update()
        self.hero.bullets.draw(self.screen)

    @staticmethod
    def __game_over():
        print("游戏结束")
        pygame.quit()
        exit()
#主函数，这样写可以被其他模块当成函数调用
if __name__ == '__main__':
    game = PlaneGame()
    game.start_game()
	
```
**plane_sprites.py** 源码：
```
import random

import pygame
from plane_main import SCREEN_RECT


class GameSprite (pygame.sprite.Sprite):
    def __init__(self, image_name, speed=1 ):
        #子类初始化函数需要先执行父类的初始化函数
        super().__init__()
        self.image = pygame.image.load(image_name)
        self.rect = self.image.get_rect()
        self.speed = speed

    def update(self):
    	  #更改图片坐标
        self.rect.y += self.speed

class Background(GameSprite):
    def __init__(self, is_alt = False):
        super().__init__("./images/background.png")
        #两个背景对象放在不同的位置，已达到无缝循环
        if is_alt:
            self.rect.y = -self.rect.height
    def update(self):
        super().update()
        #图像向下移动
        if self.rect.y >= SCREEN_RECT.height :
            self.rect.y = -self.rect.height
class Enemy(GameSprite):
    def __init__(self):
        super().__init__("./images/enemy1.png")
        #敌机速度随机
        self.speed = random.randint(1,3)
        self.rect.bottom = 0
        max_x = SCREEN_RECT.width - self.rect.width
        self.rect.x = random.randint(0,max_x)

    def update(self):
        super().update()
        #敌机飞出主窗体，释放对象
        if self.rect.y >= SCREEN_RECT.height:
            self.kill()

    def __del__(self):
       pass

class Hero(GameSprite):
    def __init__(self):
        super().__init__("./images/me1.png", 0)
        self.rect.centerx = SCREEN_RECT.centerx
        self.rect.bottom = SCREEN_RECT.bottom - 120
        #创建子弹对象
        self.bullets = pygame.sprite.Group()

    def update(self):
        self.rect.x += self.speed
        if self.rect.x < 0:
            self.rect.x = 0
        elif self.rect.right > SCREEN_RECT.right:
            self.rect.right = SCREEN_RECT.right

    def fire(self):
    	  # 三发子弹
        for i in (0, 1, 2):
            bullet = Bullet()
            #子弹的横坐标相同，纵坐标相差20
            bullet.rect.bottom = self.rect.y - i * 20
            bullet.rect.centerx = self.rect.centerx
            self.bullets.add(bullet)

class Bullet(GameSprite):
    def __init__(self):
       super().__init__("./images/bullet1.png", -2)

    def update(self):
        super().update()
        #子弹飞出主窗体，释放对象
        if self.rect.bottom < 0:
            self.kill()

    def __del__(self):
        pass
```
**代码逻辑:**
* 初始函数
	* 创建主窗体
	* 创建对象 
* 开始函数
	* 循环
	* 监听、碰撞检测、更新位置、刷新
* 对象初始函数
	* 加载图片、初始速度、初始位置
* 对象更新函数：
	* 移动位置、边界限定


## 总结

通过这个项目觉得python难点在于
 * 对于类对象和实例对象的理解
 * 程序运行环境的理解：命名空间和作用域
 * 类的属性的定义方式

# 结语
 基本语法学习结束，准备学习java的web技术，之后学习python的web技术