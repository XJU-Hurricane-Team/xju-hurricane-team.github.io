# 软件工程常用方法

我不是软件工程专业的，也没有系统的学习过软件工程。只是总结分享下我在开发过程中一些比较好用的方法。

有些概念比较抽象，需要花时间去理解。

> 参照：
> 
> [软件工程复习（一） | 建明 | Ming.J](https://sunnyqjm.github.io/2017/12/22/2017-12-22-%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B%E5%A4%8D%E4%B9%A0%EF%BC%88b%EF%BC%89/)
> 
> [C语言大师：编得狂，骂得响[前篇]_哔哩哔哩 (゜-゜)つロ 干杯~-bilibili](https://www.bilibili.com/video/av238869905/)
> 
> [C语言大师：编得狂，骂得响[后篇]_哔哩哔哩 (゜-゜)つロ 干杯~-bilibili](https://www.bilibili.com/video/av1250963900/)

---

## 表驱动法

> 参照：[表驱动法 | cchroot's blog](https://cchroot.github.io/2020/05/23/%E8%A1%A8%E9%A9%B1%E5%8A%A8%E6%B3%95/)

表驱动法可以简化判断逻辑，并且速度快，易于维护。

表驱动法的核心是表，所以也叫查表法。

举个例子，我们虽然可以通过泰勒展开计算sin 1这类非特殊角度的三角函数，但显然非常费事费力。但如果我们事先将0 ~ 90度的所有三角函数值写在一张纸上，那么得到sin 1的值就非常简单，直接查表就可以了。ARM DSP库的三角函数就是用这个方法实现的。

再举个例子，假如我有一个遥控器，有6个按键，编号`KEY0 ~ KEY5`，对应数字0~5，每个按键对应不同的功能，而且更复杂的是`KEY1, KEY2, KEY3`还作为模式切换，不同模式下某些按键的功能还不一样。假设功能表是下面的：

| 按键   | mode1        | mode2        | mode3           |
| ---- | ------------ | ------------ | --------------- |
| KEY0 | shutdown     | shutdown     | shutdown        |
| KEY1 | switch_mode1 | switch_mode1 | switch_mode1    |
| KEY2 | switch_mode2 | switch_mode2 | switch_mode2    |
| KEY3 | switch_mode3 | switch_mode3 | switch_mode3    |
| KEY4 | Play Music   | Play Radio   | Increase volume |
| KEY5 | Stop Music   | Stop Radio   | Decrease volume |

假如不用表驱动，就用`if-else, switch-case`，那么代码将会是这样：

```
void key_scan() {
    // scan the key ...
    switch (key) {
        case KEY0: {
            shutdown();
        } break;

        case KEY1: {
            mode = 1;
        } break;

        case KEY2: {
            mode = 2;
        } break;

        case KEY3: {
            mode = 3;
        } break;

        case KEY4: {
            if (mode == 1) {
                play_music();
            } else if (mode == 2) {
                play_radio();
            } else if (mode == 3) {
                increase_volume();
            }
        } break;

        case KEY5: {
            if (mode == 1) {
                stop_music();
            } else if (mode == 2) {
                stop_radio();
            } else if (mode == 3) {
                decrease_volume();
            }
        } break;

        default: {
        } break;
    }
}
```

虽然用了`switch-case`，看着也不是太复杂，假如有非常多的按键，这里的判断逻辑将会变得非常复杂。可以用数组+函数指针来简化代码：

```
void (*key_callback)(int /* key */)[5] = {shutdown,    switch_mode, switch_mode,
                                          switch_mode, play_music,  stop_music};

void switch_mode(int key) {
    if (key == KEY1) {
        key_callback[KEY4] = play_music;
        key_callback[KEY5] = stop_music;
    } else if (key == KEY2) {
        key_callback[KEY4] = play_radio;
        key_callback[KEY5] = stop_radio;
    } else if (key == KEY3) {
        key_callback[KEY4] = increase_volume;
        key_callback[KEY5] = decrease_volume;
    }
}

void key_scan() {
    // scan the key ...
    key_callback[key_press](key_press);
}
```

如果我们要切换模式，只需要修改表中的函数指针就可以了，不需要多余的判断，而且扩展非常方便。

> 如果你对函数指针不熟悉，请复习C语言基础。

如果使用Python，那么用字典就可以了，非常方便。

其他应用参照上面的链接。

## 事件处理框架

占位
