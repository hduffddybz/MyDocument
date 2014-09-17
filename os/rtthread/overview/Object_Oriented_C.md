# RT-Thread C语言对象化模型 #

## 面向对象基本特征

面向对象的三个基本特征是：封装、继承、多态。封装用以隐藏代码内部实现，继承可以用以复用现有的代码，而多态则可改变对象行为。

## C语言实现封装

## C语言实现继承

继承是指在某些情况下，一个类会有其子类，子类会比原来的父类要更加具体化（例如狗有它的子类“牧羊犬”等）。子类会继承父类的属性行为，但也可能包括他们自己的属性行为。继承增强了系统的可重用性，也增加了系统的可扩展性，用C语言实现的继承机制如下所示：

``` c
#include <stdio.h>

/* 父类 */
struct parent_class
{
    int a, b;
    char *str;
};

/* 子类 */
struct child_class
{
    struct parent_class p;
    int a, b;
};


int main()
{
    /* 子类对象和指针 */
    struct child_class obj, *obj_ptr;
    /* 父类指针 */
    struct parent_class *parent_ptr;

    obj_ptr = &obj;
    /* 取父类指针 */
    parent_ptr = (struct parent_class*)&obj;

    /* 父类属性的操作 */
    parent_ptr->a = 1;
    parent_ptr->b = 5;

    /* 子类属性的操作 */
    obj_ptr->a = 10;
    obj_ptr->b = 100;

    return 0;
}

```

## C语言实现多态

多态是指由继承产生的相关的不同的类，对象会对同一消息作出不同的响应（例如狗和鸡都会有“叫”的方法，但是调用狗的“叫（）”会吠叫，调用鸡的“叫（）”会啼叫）。利用多态性用户可发送一个通用消息，而将所有实现细节都留给接收的消息对象自行决定，C语言实现多态性的代码如下所示：

``` c
#include <stdio.h>
#include <assert.h>

/* 抽象基类 */
struct parent_class
{
    int a;

    /* 反映不同类别属性的方法 */
    void (*vfunc)(struct parent_class *self, int a);
};

/* 继承至parent_class的子类 */
struct child_class
{
    struct parent_class parent;
    int b;
};

void parent_class_vfunc(struct parent_class* self, int a)
{
    assert(self != NULL);
    assert(self->vfunc != NULL);
    self->vfunc(self, a);
}

/* 子类虚函数的实现 */
static void child_class_vfunc(struct parent_class* self, int a)
{
    struct child_class* child = (struct child_class *)self;
    child->b = a + 10;
}

/* 子类的构造函数 */
void child_class_init(struct child_class* self)
{
    struct parent_class* parent;

    /* 强制类型转换获得父类指针 */
    parent = (struct parent_class*)self;
    assert(parent != NULL);

    /* 设置子类的虚函数 */
    parent->vfunc = child_class_vfunc;
}

int main()
{
    struct child_class obj, *objptr;
    struct parent_class *parent_ptr;

    objptr = &obj;
    objptr->b = 10;
    child_class_init(objptr);

    /*多态例子*/
    parent_ptr = (struct parent_class*)objptr;
    parent_ptr->vfunc(parent_ptr, 10);
    printf("b = %d\n", objptr->b);
}
```

## RT-Thread 面向对象模型实例

以下以device、serial device为例说明：

首先看下device结构体的定义（见include/rtdef.h）

``` c
struct rt_device
{
    struct rt_object          parent;                   /**< inherit from rt_object */

    enum rt_device_class_type type;                     /**< device type */
    rt_uint16_t               flag;                     /**< device flag */
    rt_uint16_t               open_flag;                /**< device open flag */

    rt_uint8_t                ref_count;                /**< reference count */
    rt_uint8_t                device_id;                /**< 0 - 255 */

    /* device call back */
    rt_err_t (*rx_indicate)(rt_device_t dev, rt_size_t size);
    rt_err_t (*tx_complete)(rt_device_t dev, void *buffer);

    /* common device interface */
    rt_err_t  (*init)   (rt_device_t dev);
    rt_err_t  (*open)   (rt_device_t dev, rt_uint16_t oflag);
    rt_err_t  (*close)  (rt_device_t dev);
    rt_size_t (*read)   (rt_device_t dev, rt_off_t pos, void *buffer, rt_size_t size);
    rt_size_t (*write)  (rt_device_t dev, rt_off_t pos, const void *buffer, rt_size_t size);
    rt_err_t  (*control)(rt_device_t dev, rt_uint8_t cmd, void *args);

    void                     *user_data;                /**< device private data */
};
```

结构体最前面字段struct rtobject parent说明了rtdevice对象继承了rtobject的对象，体现了面向对象中的继承关系。rtdevice对象中还定义了一组函数指针，可用以实现面向对象中的多态，如在rt_hw_serial_register()函数（在components/drivers/serial/serial.c）中实现的以下代码：

``` c
rt_err_t rt_hw_serial_register(struct rt_serial_device *serial,
                               const char              *name,
                               rt_uint32_t              flag,
                               void                    *data)
{
    struct rt_device *device;
    RT_ASSERT(serial != RT_NULL);

	/* 获得父指针 */
    device = &(serial->parent);

    device->type        = RT_Device_Class_Char;
    device->rx_indicate = RT_NULL;
    device->tx_complete = RT_NULL;

	/* 设置子类的虚函数 */
    device->init        = rt_serial_init;
    device->open        = rt_serial_open;
    device->close       = rt_serial_close;
    device->read        = rt_serial_read;
    device->write       = rt_serial_write;
    device->control     = rt_serial_control;
    device->user_data   = data;

    /* register a character device */
    return rt_device_register(device, name, flag);
}

```