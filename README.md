#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>

int main() {
    int fd;
    struct v4l2_capability cap;
    struct v4l2_format fmt;
    unsigned int i, num_buffers;
    void **buffers;

    // Open the camera device
    fd = open("/dev/video0", O_RDWR | O_NONBLOCK, 0);
    if (fd == -1) {
        perror("Open device");
        return 1;
    }

    // Query the capabilities of the device
    if (ioctl(fd, VIDIOC_QUERYCAP, &cap) == -1) {
        perror("Query capabilities");
        return 1;
    }

    // Set the video format
    CLEAR(fmt);
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    fmt.fmt.pix.width = 640;
    fmt.fmt.pix.height = 480;
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
    fmt.fmt.pix.field = V4L2_FIELD_ANY;

    if (ioctl(fd, VIDIOC_S_FMT, &fmt) == -1) {
        perror("Set format");
        return 1;
    }

    // Request the number of buffers
    num_buffers = 4;
    if (ioctl(fd, VIDIOC_REQBUFS, &num_buffers) == -1) {
        perror("Request buffers");
        return 1;
    }

    // Map the buffers
    buffers = calloc(num_buffers, sizeof(void *));
    for (i = 0; i < num_buffers; i++) {
        struct v4l2_buffer buf;
        void *mem;

        mem = malloc(fmt.fmt.pix.sizeimage);
        if (!mem) {
            perror("Out of memory");
            return 1;
        }

        buffers[i] = mem;

        CLEAR(buf);
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_USERPTR;
        buf.index = i;
        buf.m.userptr = (unsigned long)mem;
        buf.length = fmt.fmt.pix.sizeimage;

        if (ioctl(fd, VIDIOC_QBUF, &buf) == -1) {
            perror("Queue buffer");
            return 1;
        }
    }

    // Capture 10 frames of video
    for (i = 0; i < 10; i++) {
        struct v4l2_buffer buf;
        fd_set fds;
        struct timeval tv;
        int r;

        CLEAR(buf);
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_USERPTR;

        if (ioctl(fd, VIDIOC_DQBUF, &buf) == -1) {
            perror("Dequeue buffer");
            return 1;
        }

        // Process the captured buffer here

        if (ioctl(fd, VIDIOC_QBUF, &buf) == -1) {
            perror("Queue buffer");
            return 1;
