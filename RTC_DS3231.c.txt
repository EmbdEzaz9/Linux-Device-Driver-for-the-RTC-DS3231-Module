#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/i2c.h>
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/bcd.h>
#include <asm/errno.h>

MODULE_LICENSE("GPL");

typedef struct _ds3231_time {
    u8 second;
    u8 minute;
    u8 hour;
} ds3231_time_t;

//static struct i2c_adapter *adapter;
static struct i2c_client *ds3231_client;

static dev_t ds3231;
static struct class *ds3231_class;
static struct cdev device_ds3231;

static ssize_t ds3231_read(struct file *file, char __user *user_buffer, size_t count, loff_t *offset);
static ssize_t ds3231_write(struct file *file, const char __user *user_buffer, size_t count, loff_t *offs);

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .llseek = no_llseek,
    .read = ds3231_read,
    .write = ds3231_write,
};

static const struct i2c_device_id ds3231_id[] = {
    {"ds3231", 0},
    {}
};

MODULE_DEVICE_TABLE(i2c, ds3231_id);

static struct i2c_driver ds3231_driver = {
    .driver = {
        .owner = THIS_MODULE,
        .name = "ds3231",//ds3231_drv_v
    },
};

static struct i2c_board_info ds3231_info = {
    I2C_BOARD_INFO("ds3231", 0x68)
};

int ds3231_init(void)
{
    const struct i2c_board_info info = {I2C_BOARD_INFO("ds3231", 0x68)};
        struct i2c_adapter *adapter;
        int rval;
        pr_info("ds3231: initializing hardware ...\n");
        ds3231_client = NULL;
        adapter = i2c_get_adapter(1);
            if (adapter == NULL) {
         pr_err("ds3231: i2c adapter not found\n");
         return -1;
         }
        ds3231_client = i2c_new_client_device(adapter, &info);
        if (ds3231_client == NULL) {
                 pr_err("ds3231: failed to register i2c device\n");
                 return -1;
        }
        rval = i2c_add_driver(&ds3231_driver);
        if (rval < 0) {
                 pr_err("ds3231: failed to ds3231 i2c driver\n");
                 i2c_unregister_device(ds3231_client);
                         ds3231_client = NULL;
        }
        pr_info("ds3231: hardware initialization completed.\n");
            return rval;
}

void ds3231_exit(void)
{
   pr_info("ds3231: uninitializing hardware ...\n");
        if (ds3231_client != NULL)
        {
                i2c_del_driver(&ds3231_driver);
                        i2c_unregister_device(ds3231_client);
                                ds3231_client = NULL;
        }
}

int ds3231_io_init(void){
        int ret;
        //allocate the device number
         ret = alloc_chrdev_region(&ds3231, 0, 1, "ds3231");
        if(ret<0){
                pr_err("ds3231: alloc_chrdev_region() failed\n");
                return ret;
        }
        //initialize the device file
        cdev_init(&device_ds3231, &fops);
		
        // register to  device and kernel;
         if (cdev_add(&device_ds3231, ds3231, 1) < 0)
        {
                 pr_err("ds3231: character device could not be registered");
                 goto unreg_chrdev;
         }
        // class create
        ds3231_class = class_create(THIS_MODULE, "chardev");

         if (ds3231_class == NULL)
         {
                 pr_err("ds3231: character device class could not be created\n");
               goto clenup_cdev;
         }
        //device file
        if (device_create(ds3231_class, NULL, ds3231, NULL, "ds3231") == NULL)
         {
                 pr_err("ds3231: character device could not be created\n");
                 goto cleanup_chrdev_class;
         }
        return 0;

cleanup_chrdev_class:
    class_destroy(ds3231_class);
clenup_cdev:
    cdev_del(&device_ds3231);
unreg_chrdev:
    unregister_chrdev_region(ds3231, 1);
return -1;
}

void ds3231_io_exit(void){
        device_destroy(ds3231_class, ds3231);
    class_destroy(ds3231_class);
    cdev_del(&device_ds3231);
    unregister_chrdev_region(ds3231, 1);
    pr_info("ds3231: unloaded chacter device driver\n");
}

int ds3231_write_time(ds3231_time_t *time)
{
    u8 secs, mins, hrs;
    int retval;

    secs = ((time->second / 10) << 4) | (time->second % 10);
    mins = ((time->minute / 10) << 4) | (time->minute % 10);
    hrs = ((time->hour / 20) << 5) | (((time->hour % 20) / 10) << 4) | ((time->hour % 20) % 10);

    retval = i2c_smbus_write_byte_data(ds3231_client, 0x00, secs);
    if (retval < 0) {
        pr_err("Error setting seconds: %d\n", retval);
        return retval;
    }

    retval = i2c_smbus_write_byte_data(ds3231_client, 0x01, mins);
    if (retval < 0) {
        pr_err("Error setting minutes: %d\n", retval);
        return retval;
    }

    retval = i2c_smbus_write_byte_data(ds3231_client, 0x02, hrs);
    if (retval < 0) {
        pr_err("Error setting hours: %d\n", retval);
        return retval;
    }
    return 0; // Return success if all writes were successful
}

void read_ds3231_time(struct i2c_client *ds3231_client, ds3231_time_t *time)
{
    int sec, min, hour;

    sec = i2c_smbus_read_byte_data(ds3231_client, 0x00);
    min = i2c_smbus_read_byte_data(ds3231_client, 0x01);
    hour = i2c_smbus_read_byte_data(ds3231_client, 0x02);

    if (sec < 0 || min < 0 || hour < 0) {
        pr_err("Error reading time from DS3231\n");
        return;
    }

    sec = bcd2bin(sec & 0x7F);
    min = bcd2bin(min);
    hour = bcd2bin(hour & 0x3F);

    time->second = sec;
    time->minute = min;
    time->hour = hour;
}

int ds3231_open(struct inode *inode, struct file *file)
{
    pr_debug("ds3231: opened character device\n");
    return 0;
}

ssize_t ds3231_read(struct file *file, char __user *user_buffer, size_t count, loff_t *offset)
{
    char buffer[30];
    int to_copy, not_copy;
    ds3231_time_t time;

    read_ds3231_time(ds3231_client, &time);

    memset(buffer, '\0', sizeof(buffer));
    snprintf(buffer, sizeof(buffer), "%02d:%02d:%02d", time.hour, time.minute, time.second);

    to_copy = min(count, sizeof(buffer));
    not_copy = copy_to_user(user_buffer, buffer, to_copy);

    if (not_copy != 0) {
        return -EFAULT;
    } else {
        return to_copy;
    }
}

static ssize_t ds3231_write(struct file *file, const char __user *user_buffer, size_t count, loff_t *offs)
{
    char buffer[30];
    int not_copy;
    ds3231_time_t time;
s32 hour, minute, second;
    if (count > sizeof(buffer)) {
        return -EINVAL;
    }

    not_copy = copy_from_user(buffer, user_buffer, count);
    if (not_copy != 0) {
        return -EFAULT;
    }

     if (sscanf(buffer, " %d:%d:%d",&hour,&minute,&second) < 0)
         {
                 return -1;
         }
     if ((23 < hour || 0 > hour) || (59 < minute || 0 > minute) || (59 < second || 0 > second))
         {
                       return -1;
         }

         time.hour = hour;
         time.minute = minute;
         time.second = second;
    
	pr_err("ds3231: write time  %d:%d:%d\n", time.hour, time.minute, time.second);
    ds3231_write_time(&time);

    return count;
}

int ds3231_close(struct inode *inode, struct file *file)
{
    pr_debug("ds3231: closed character device\n");
    return 0;
}

static int __init moduleinit(void)
{
   int rval = ds3231_init();
        if (rval < 0) {
                 return rval;
        }
         rval = ds3231_io_init();
         if (rval < 0) {
                  ds3231_exit();
         }
         return rval;
}

static void __exit moduleexit(void)
{
    ds3231_io_exit();
    ds3231_exit();
}

module_init(moduleinit);
module_exit(moduleexit);
