**Table Of Contents**

[1 INTRODUCTION](#1-introduction)<br/>
&emsp;[1.1 SPECIFICATION](#11-specification)<br/> 
&emsp;&emsp;[1.1.1 RESOURCE UTILIZATION](#111-resource-utilization)<br/>
&emsp;&emsp;[1.1.2 BANDWIDTH CONSUMPTION](#112-bandwidth-consumption)<br/>
&emsp;[1.2 PERFORMANCE FOR VARIOUS RESOLUTIONS](#12-performance-for-various-resolutions)<br/>
&emsp;[1.3 BENCHMARK](#13-benchmark)<br/>
[2 EVALUATION](#2-evaluation)<br/>
&emsp;[2.1 AWS F1 INSTANCE](#21-aws-f1-instance)<br/>
&emsp;&emsp;[2.1.1 INSTALLATION](#211-installation)<br/>
&emsp;&emsp;[2.1.2 HOW TO EVALUATE](#212-how-to-evaluate)<br/>
&emsp;[2.2 AWS Marketplace](#22-aws-marketplace)<br/>
&emsp;[2.3 ALVEO U50](#22-alveo-u50)<br/>
&emsp;&emsp;[2.3.1 GETTING A CREDENTIAL FILE](#221-getting-a-credential-file)<br/>
&emsp;&emsp;[2.3.2 INSTALLATION](#222-installation)<br/>
&emsp;&emsp;[2.3.3 HOW TO EVALUATE](#223-how-to-evaluate)<br/>
[3 DUAL-KERNEL PERFORMANCE IN AWS EC2 F1 INSTANCE](#3-dual-kernel-performance-in-aws-ec2-f1-instance)<br/>
[4 REST API](#4-rest-api)<br/>
[5 CHANGE LOG](#5-changelog)<br/>
[6 CONTACT US](#6-contact-us)

# 1 INTRODCUTION
The Pulsar-VMAF is an hardware accelerator to measure VMAF(Video Multi-Method Assessment Fusion) for a
perceptual video quality assessment. The accelerator is available for the following platforms:
- Xilinx Alveo U50
- AWS F1(f1.2xlarge)   
> Note. The accelerator can be also ported to the other Alveo cards like U200 and U250, and the cloud platforms like Azure NPVM.

The following diagram shows how the accelerator can be used in a transcoding process:

<img src="images/how_the_accelerator_work.png" width="70%" height="70%">

In one U50 or f1.2xlarge, two kernels are instantiated. Each kernel consists of hardware blocks for scaling and VMAF measurement.
- The algorithm of the scaler is the same as the bicubic scaling in ffmpeg. The scaler is typically used for scaling the distorted(e.g., transcoded) video of which resolution is different from the original video prior to the measurement of VMAF score.
- Input format of the kernel is YUV. The VMAF uses only luminance samples. Therefore, any of 4:2:0, 4:2:2:, and 4:4:4 can be used as an input.
- We integrated the accelerator in ffmpeg. So, any video coding format which is decodable in ffmpeg can be handled.

## 1.1 SPECIFICATION

We accelerated VMAF v2.0.0 built without any CPU acceleration feature. Details of the VMAF can be found in [https://github.com/Netflix/vmaf](https://github.com/Netflix/vmaf).

---
> Note. The latest version of VMAF is v2.2.0 released on Jul. 3, 2021.   
> The updates from v2.0.0 we accelerated is not related to the algorithm to derive VMAF score.

---

The following table shows the specification of Pulsar-VMAF targeted to Alveo U50 and AWS F1:

<table>
    <thead>
        <tr>
            <th></th>
            <th>Alveo U50</th>
            <th>AWS f1.2xlarge</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Min. resolution of input video</td>
            <td colspan="2" align="center"> 160x120 </td>
        </tr>
        <tr>
            <td>Max. resolution of input video</td>
            <td colspan="2" align="center"> 3840x2160 </td>
        </tr>
        <tr>
            <td>Bitdepth</td>
            <td colspan="2" align="center"> 8-bit </td>
        </tr>
        <tr>
            <td>Scaling input video prior to measuring VMAF</td>
            <td colspan="2"> A hardware scaler is integrated to upscale the distorted video to the same resolution of the reference video prior to measurement of VMAF. </td>
        </tr>
        <tr>
            <td>Input video format</td>
            <td colspan="2" align="center"> Compressed raw </td>
        </tr>
        <tr>
            <td>Number of kernels in FPGA</td>
            <td colspan="2" align="center"> 2 </td>
        </tr>
        <tr>
            <td>Operating clock frequency</td>
            <td align="center">235MHz</td>
            <td align="center">219MHz</td>
        </tr>
        <tr>
            <td>Performance(with compressed videos)*</td>
            <td align="center">3840x2160 74fps/kernel</td>
            <td align="center">3840x2160 67fps/kernel</td>
        </tr>
        <tr>
            <td>Performance(with raw videos)*</td>
            <td align="center">3840x2160 75fps/kernel</td>
            <td align="center">3840x2160 70fps/kernel</td>
        </tr>
        <tr>
            <td>Shell</td>
            <td align="center">Xilinx_u50_gen3x16_xdma_201920_3</td>
            <td align="center">xilinx_aws-vu9p-f1_shellv04261818_201920_2</td>
        </tr>
        <tr>
            <td>Power consumption</td>
            <td align="center">43.123W</td>
            <td align="center">65.836W</td>
        </tr>
    </tbody>
</table>

**\* Environment used for the evaluation:**
- Alveo U50: tested using Intel® Core™ i9-9900K @ 3.6GHz, 64G(DDR4 16GB PC4-21300 x 4), CentOS Linux Release 7.8.2003
- AWS f1.2xlarge: You can find more details of F1 instance in [https://aws.amazon.com/ec2/instance-types/](https://aws.amazon.com/ec2/instance-types/).  
- The performance was measured using the VMAF accelerator integrated in ffmpeg for both compressed and raw video cases.
- ffmpeg version: git-2021-02-18-034e6e2

### 1.1.1 Resource utilization
Utilization on AWS (VU9P), per kernel
  
| LUT | Register | DSP | BRAM36 | URAM |
|----:|---------:|----:|-------:|-----:|
| 192,121 | 358,995 | 1,927 | 383 | 56 |

### 1.1.2 Bandwidth consumption
- Condition : 4K@65fps (3840x2160) 
- Between the kernel and the memory inside the FPGA card, per kernel

| Read BW [GBytes/s] | Write BW [GBytes/s] |
|----------------:|----------------------:|
| 3.74 | 2.66  |

## 1.2 Performance for various resolutions
The following table shows the performance in fps for various resolutions with a single kernel activated:

|             | Alveo U50 [fps] | AWS f1.2xlarge [fps]  |
|------------:|----------------:|----------------------:|
|  3840x2160  | 74              | 67                    |
|  2560x1440  | 155             | 112                   |
|  1920x1080  | 248             | 170                   |
|  1280x720   | 455             | 308                   |
|  640x360    | 988             | 686                   |
|  160x120    | 1,646           | 1,163                 |
- Total frames: 3840x2160 150 frames x 100 repetitions with -stream_loop 99
- Input format: H.264 compressed
## 1.3 Benchmark
The following graph shows performances of the software on various EC2 instances and the hardware to measure VMAF.
- c5.12xlarge
    + 48 vCPUs of 2nd generation Intel® Xeon™ Scalable Processors (Cascade Lake) 3.6GHz, 96GiB memory
    + $2.04/hour (on-demand)
- c5.24xlarge
    + 96 vCPUs of 2nd generation Intel® Xeon™ Scalable Processors (Cascade Lake) 3.6GHz, 96GB memory
    + $4.608/hour (on-demand)
- AWS f1.2xlarge
    + 8 vCPUs of Intel® Xeon™ E5-2686 v4 (Broadwell) processors, 122 GiB memory
    + Xilinx Virtex UltraScale+ VU9P FPGA
    + f1.2xlarge: $1.65/hour (on-demand)
- 3840x2160 150 frames (H.264 compressed), 100 repetitions
- Only one of two VMAF kernels in the FPGA is activated.  
<br/>
<img src="images/benchmark_speed.png" width="70%" height="70%">
<br/>
- In the case of “f1.2xlarge SW (n_threads=8)”, only CPU in the instance is used to measure VMAF score without hardware acceleration.

# 2 EVALUATION
This chapter gives how to install and evaluate the Pulsar-VMAF accelerator using AWS F1 instance and Alveo U50. 

## 2.1 AWS F1 INSTANCE 
For the evaluation, we provide a specific AMI(Amazon Machine Image) and AFI(Amazon FPGA Image).  
<br/>
<img src="images/fpga_acceleration_using_f1.png" width="70%" height="70%">
<br/>

### 2.1.1 INSTALLATION
#### REQUIREMENT

AWS IAM Account
- In order to allow a user to use the accelerator, AWS IAM account in 12-digit number is required.  
A user needs to send an email with the information to info@blue-dot.io.  It’ll be notified when permission is set for the evaluation.

It’s required also to inform if a user want to use the other region than Virginia for the EC2 F1 instance.

#### PROCEDURE

##### Step 1. Choose the AMI we sahred
In the first step of the “Launch instance” in AWS, you have to choose “Pulsar-VMAF.evaluation” AMI we shared.  
Please contact us if you cannot see the AMI in a list of available AMIs.
<br/>
<img src="images/aws_f1_choose_ami.png" width="70%">
<br/>

##### Step 2. Choose f1.2xlarge instance
choose f1.2xlarge
<br/>
<img src="images/aws_f1_choose_f1.2xlarge.png" width="70%">
<br/>

##### Step 3. Review and launch the instance
Launch the instance you created after review and further configuration if necessary.
##### Step 4. Connect to your instance
There are a few ways to connect the instance. Here is an example for the connection to an instance using a private key file, specified when creating the instance, and its public DNS.
```bash
ssh -i “your_keypair.pem” ec2-user@ec2-12-345-678-901.compute-1.amazonaws.com
```
You should replace the your_keypair.pem and the ce2-12-345-678-901.copmpute-1.amazonaws.com with yours.
If it’s correctly created an instance and connected to it, you can see the following messages:
```

 _______  __   __  ___      _______  _______  ______           __   __  __   __  _______  _______
|       ||  | |  ||   |    |       ||   _   ||    _ |         |  | |  ||  |_|  ||   _   ||       |
|    _  ||  | |  ||   |    |  _____||  |_|  ||   | ||   ____  |  |_|  ||       ||  |_|  ||    ___|
|   |_| ||  |_|  ||   |    | |_____ |       ||   |_||_ |____| |       ||       ||       ||   |___
|    ___||       ||   |___ |_____  ||       ||    __  |       |       ||       ||       ||    ___|
|   |    |       ||       | _____| ||   _   ||   |  | |        |     | | ||_|| ||   _   ||   |
|___|    |_______||_______||_______||__| |__||___|  |_|         |___|  |_|   |_||__| |__||___|

                                                                             https://blue-dot.io
                                                                                info@blue-dot.io

#### HOWTO ####
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=/usr/share/libbdvmaf/vmaf_4k_v0.6.1.json:kernel_path=/usr/share/libbdvmaf/f1_binary.xclbin:log_path=test.xml:device=f1 -f null -
You can run two ffmpegs in a f1.2xlarge, four ffmpegs in a f1.4xlarge
```
The list of files is as follows at the initial connection:
- 2160_dst.mp4 - distorted video of which quality is to be measured, playback time: 6sec @ 25fps
- 2160.mp4 - reference video
- vmaf_4k_v0.6.1.json, vmaf_v0.6.1.json - VMAF models
    + Refer to [https://github.com/Netflix/vmaf/blob/master/resource/doc/models.md](https://github.com/Netflix/vmaf/blob/master/resource/doc/models.md) for more details.
- f1_binary.xclbin - AWS FPGA binary

### 2.1.2 HOW TO EVALUATE
In one f1.2xlarge instance, two kernels are instantiated. The following examples show how to run one kernel and two kernels, respectively.  
In the AMI installed, two video bistreams, 2160.mp4 and 2160_dst.mp4, are provided for examples.
#### Measuring VMAF of compressed videos using one kernel
```bash
ffmpeg -stream_loop 99 -i 2160_dst.mp4 -vsync 0 -stream_loop 99 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin:device=f1 -f null -
```
- -stream_loop: an option to specify the number of repetition (you can remove it). In the above command, the video streams are decoded 100 times.

When ffmpeg finishes, a fps and an average VMAF score is reported on the screen.
```
frame=15000 fps= 67 q=-0.0 Lsize=N/A time=00:10:00.00 bitrate=N/A speed=2.67x
video:7852kB audio:0kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: unknown
[Parsed_libbdvmaf_0 @ 0x759d0c0] Total number of processed frames: 15000
[Parsed_libbdvmaf_0 @ 0x759d0c0] AVG. VMAF Score: 87.467400
```
If you want to output scores of models per frame to a file, use "log_path" and "log_fmt" options as the following:
```bash
ffmpeg -stream_loop 99 -i 2160_dst.mp4 -vsync 0 -stream_loop 99 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin:log_path=log.json:log_fmt=json:device=f1 -f null -
```
The supported output formats are json, xml and csv.

The following is an example of output with "log_fmt=json" option.
```
{
  "version": "2.2.0 based H/W",
  "frames": [
    {
      "frameNum": 0,
      "metrics": {
        "integer_adm2": 1.000000,
        "integer_adm_scale0": 1.000000,
        "integer_adm_scale1": 1.000000,
        "integer_adm_scale2": 1.000000,
        "integer_adm_scale3": 1.000000,
        "integer_motion2": 0.000000,
        "integer_motion": 0.000000,
        "integer_vif_scale0": 1.000000,
        "integer_vif_scale1": 1.000000,
        "integer_vif_scale2": 1.000000,
        "integer_vif_scale3": 1.000000,
        "vmaf": 100.000000
      }

...

  "pooled_metrics": {
    "integer_adm2": {
      "min": 0.858771,
      "max": 1.062500,
      "mean": 0.954343,
      "harmonic_mean": 0.953602
    },
    "integer_adm_scale0": {
      "min": 0.979701,
      "max": 1.023748,
      "mean": 0.999715,
      "harmonic_mean": 0.999673
    },
    "integer_adm_scale1": {
      "min": 0.915208,
      "max": 1.059503,
      "mean": 0.973407,
      "harmonic_mean": 0.972973
    },
...
```

---
**Note**
1. When you specify a reference and a distorted video in ffmpeg command line options, please place the
distorted video first. It’s a requirement of ffmpeg.
2. For the first run after your connection to the instance, a little bit lower fps could be reported due to
loading time of AFI(Amazon FPGA Image) into FPGA. The AFI is not loaded for the next runs.

---
#### Measuring VMAF of raw videos using one kernel
The following example shows how to convert a compressed video to a raw video in YUV format.
```bash
ffmpeg -i 2160.mp4 2160.yuv
ffmpeg -i 2160_dst.mp4 2160_dst.yuv
```
Run the following command to measure VMAF score of those raw videos.   
```bash
ffmpeg -stream_loop 99 -pix_fmt yuv420p -s 3840x2160 -i 2160_dst.yuv -stream_loop 99 -s 3840x2160 -pix_fmt yuv420p -i 2160.yuv -lavfi libbdvmaf=model_path=vmaf_v0.6.1.json:kernel_path=f1_binary.xclbin:shortest=1:device=f1 -f null -
```
The same score as the case of compressed videos is reported, and the speed is a little bit better due to less computation load on the CPUs.
#### Running two kernels
```bash
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin:device=f1 -f null -
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin:device=f1 -f null -
```
## 2.2 AWS Marketplace
You can also evaluate Pulsar-VMAF trial version in AWS marketplace.

Visit https://aws.amazon.com/marketplace/pp/prodview-j2x24yjmlehjw.

<img src="images/pulsar-vmaf-aws-marketplace.png" width="50%">

For instruction guide, please refer to [2.1.2 HOW TO EVALUATE](#212-how-to-evaluate)

## 2.3 ALVEO U50
Pulsar-VMAF supports on-premis environment using Xilinx Alveo U50 card.
For more information about Xilinx Alveo U50 Accelator card, please visit the Xilinx website.
[https://www.xilinx.com/products/boards-and-kits/alveo/u50.html](https://www.xilinx.com/products/boards-and-kits/alveo/u50.html)

### 2.3.1 GETTING A CREDENTIAL FILE
#### Creating account
If you have no account for [Xilinx appstore](https://appstore.xilinx.com/), please visit the site.
You can see the following sing-up page, then click "Create Account".

<img src="images/xilinx_appstore_signup.png" width="20%">

#### Creating an access key and downloading it
Login to the site and click "Access keys" in the left menu pane.

<img src="images/xilinx_appstore_menu.png" width="20%">

Then click "Create an Access Key" in the top of the page.

<img src="images/xilinx_appstore_creating_key.png" width="20%">

Now you can see the following popup window for a new credential and click "Download JSON" button.
You can find the "cred.json" in your download folder.

<img src="images/xilinx_appstore_access_key.png" width="20%">


#### Contact us

Please email <info@blue-dot.io> for us to share the Pulsar-VMAF to you.  

YOU CAN MOVE ON NEXT STEP AFTER GETTING A CONFIRMATION EMAIL.

---
:information_source:  
Pulsar-VMAF is not regitered to Xilinx Appstore yet but in progress.<br/>
Until finishing the work, we need to give your account permission to find the product in Xilinx Appstore.

---

#### Requesting free trial
Click "Store" menu in the left pane and type "vmaf" in the "Product Name" field.
You can see Vmaf product. Click "SELECT" button.

<img src="images/xilinx_appstore_find_vmaf.png" width="40%">



### 2.3.2 INSTALLATION

---
**Note**  
CentOS 7 is **HIGHLY** recommended for your easy installation.  
We are preparing the software packages for other linux distributions.

---

- Install XRT runtime library from (XRT Github)(https://github.com/Xilinx/XRT)
- Install the latest software package using the following commands:
```bash
sudo yum-config-manager --add-repo https://tech.accelize.com/rpm/accelize_stable.repo
sudo yum install -y libaccelize-drm
sudo yum-config-manager --add-repo http://bluedot-yum-repo.s3-website-us-east-1.amazonaws.com/bluedot-yum.repo
sudo yum install epel-release
sudo yum install libbdvmaf-drm
sudo yum install ffmpeg-bluedot-vmaf
```
> :information_source: **Your system may require more rpm packages such as libXv, libav, etc for ffmpeg**

### 2.3.3 HOW TO EVALUATE
#### Setting up Xilinx runtime environment
The Pulsar-VMAF library(libbdvmaf) uses Xilinx XRT libraries and environment variables.<br/> 
Before evaluating it, please run the following command line.
```bash
source /opt/xilinx/xrt/setup.sh
```

#### Executing DRM manager
The DRM manager activates Pulsar-VMAF with conf.json and cred.json which you downloaded from the Xilinx appstore.<br/>
You should copy conf.json and cred.json in the working directory beacuse the DRM manager will find them there.

Please open your terminal for the DRM manager.
```bash
cp /usr/share/libbdvmaf/conf.json .
cp <your_download_folder>/cred.json .
bddrm.exe /usr/share/libbdvmaf/u50_binary.xclbin
```
You can see the following messsages if your credential file is valid.
```bash
INFO: Found 1 platforms
INFO: Selected platform 0 from Xilinx
INFO: Found 1 devices
Selected xilinx_u50_gen3x16_xdma_201920_3 as the target device
INFO: loading xclbin Kernels/u50_binary.xclbin
[DRMLIB] Start Session ..
[  info  ] 28862 , DRM session D72578773F2BE82C created.

To stop the DRM manager, please press CTRL+C.
```

#### Measuring VMAF of compressed videos using one kernel
Open another terminal then type the following command line.
```bash
ffmpeg -i <original_video_path> -i <transcoded_video_path> \
    -lavfi "[0:v]setpts=PTS-STARTPTS[reference]; \
            [1:v]setpts=PTS-STARTPTS[distorted]; \
            [distorted][reference]libbdvmaf=log_fmt=xml:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:log_path=libbdvamf.xml:model_path=vmaf_v0.6.1.json:device=f1" \
    -f null -
```
- <original_video_path>: a video file as reference one
- <transcoded_video_path>: a transcoded video file from the original video file. If you have no transcoded one you can create it easily using ffmpeg as the following.
```bash
ffmpeg -i <original_video_path> -c:v libx264 -preset ultrafast -c:a copy transcoded_video.mp4
```

---
**Note**
1. When you specify a reference and a distorted video in ffmpeg command line options, please place the
distorted video first. It’s a requirement of ffmpeg.

---
#### Measuring VMAF of raw videos using one kernel
The following example shows how to convert a compressed video to a raw video in YUV format.
```bash
ffmpeg -i <original_video_path> reference.yuv
ffmpeg -i <transcoded_video_path> distorted.yuv
```
Run the following command to measure VMAF score of those raw videos.   
```bash
ffmpeg -video_size 3840x2160 -r 24 -pixel_format yuv420p -i reference.yuv \
    -video_size 3840x2160 -r 24 -pixel_format yuv420p -i distorted.yuv \
    -lavfi "[0:v]setpts=PTS-STARTPTS[reference]; \
            [1:v]setpts=PTS-STARTPTS[distorted]; \
            [distorted][reference]libbdvmaf=log_fmt=xml:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:log_path=libbdvamf.xml:model_path=vmaf_v0.6.1.json:device=f1" \
    -f null -
ffmpeg -pix_fmt yuv420p -s 3840x2160 -i distorted.yuv -s 3840x2160 -pix_fmt yuv420p -i reference.yuv -lavfi libbdvmaf=model_path=/usr/share/libbdvmaf/vmaf_v0.6.1.json:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:shortest=1:device=f1 -f null -
```
The same score as the case of compressed videos is reported, and the speed is a little bit better due to less computation load on the CPUs.
#### Running two kernels
```bash
ffmpeg -i 2160_1.mp4 -i 2160_1_dst.mp4 -i 2160_2.mp4 -i 2160_2_dst.mp4\
    -lavfi "[0:v]setpts=PTS-STARTPTS[reference1]; \
            [1:v]setpts=PTS-STARTPTS[distorted1]; \
            [distorted1][reference1]libbdvmaf=log_fmt=xml:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:log_path=libbdvamf1.xml:model_path=vmaf_v0.6.1.json:device=f1" \
            -f null - \
    -lavfi "[2:v]setpts=PTS-STARTPTS[reference2]; \
            [3:v]setpts=PTS-STARTPTS[distorted2]; \
            [distorted2][reference2]libbdvmaf=log_fmt=xml:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:log_path=libbdvamf2.xml:model_path=vmaf_v0.6.1.json:device=f1" \
            -f null - 
```

#### Distorted resolution different from reference resolution
In case that the resolution of distorted frame is smaller than reference frame's,
Pulsar-VMAF resize a distorted frame same as resolution of a refrence frame's.<br/>
In opposite way, you need to resize the distorted frame or the reference frame using ffmpeg's scaler.
<br/>
Ex> reference: 3840x2160, distorted : 1920x1080
```bash
ffmpeg -i 2160.mp4 -i 1080.mp4 \
    -lavfi "[0:v]setpts=PTS-STARTPTS[reference]; \
            [1:v]setpts=PTS-STARTPTS[distorted]; \
            [distorted][reference]libbdvmaf=log_fmt=xml:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:log_path=libbdvamf.xml:model_path=vmaf_v0.6.1.json:device=f1" \
    -f null -
```
<br/>
Ex> reference: 1920x1080, distorted : 3840x2160
```bash
./ffmpeg -i 1080.mp4 -i 2160.mp4 \
    -lavfi "[0:v]setpts=PTS-STARTPTS[reference]; \
            [1:v]scale=1920:1080:flags=bicubic,setpts=PTS-STARTPTS[distorted]; \
            [distorted][reference]libbdvmaf=log_fmt=xml:kernel_path=/usr/share/libbdvmaf/u50_binary.xclbin:log_path=libbdvamf.xml:model_path=vmaf_v0.6.1.json:device=f1" \
    -f null -
```

# 3 DUAL-KERNEL PERFORMANCE IN AWS EC2 F1 INSTANCE
In case that VMAF is measured for raw videos, the accelerator shows almost the best performance because there
isn’t much computing load on the CPUs of the EC2 instance. But, when input videos are in compressed formats,
computing power of the CPUs is critical in the overall performance. EC2 f1.2xlarge has 8 vCPUs of Intel® Xeon™
E5-2686 v4 (Broadwell) processors and its computing power would not be enough to decode two 4K bitstreams per
kernel – in total, four 4K bitstreams for two kernels.  

The following table shows how CPU’s performance is critical in the overall performance in EC2 F1 instances:

<table>
  <thead>
    <tr>
      <th>Input video (4k)</th>
      <th>#Kernels running</th>
      <th>f1.2xlarge (fps)</th>
      <th>f1.4xlarge (fps)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan='2'>Raw (YUV)</td>
      <td>1</td>
      <td>72</td>
      <td>72</td>
    </tr>
    <tr>
      <td>2(kernel #0 / kernel #1)</td>
      <td>71/71</td>
      <td>72/72 *</td>
    </tr>
    <tr>
      <td rowspan='2'>Compressed (H.264)</td>
      <td>1</td>
      <td>67</td>
      <td>70</td>
    </tr>
    <tr>
      <td>2(kernel #0 / kernel #1)</td>
      <td>45/46</td>
      <td>64/65 *</td>
    </tr>
  </tbody>
</table>
* Two kernels in one of two FPGAs are running but 16 vCPUs are used.

As shown in the table above, an instance which has more CPUs gives higher performance. It’s also predictable that
compressed videos with higher complexity which requires higher computing power would bring a drop of the
performance. Of course, there will be the similar phenomenon in the case of the software-based VMAF
measurement application running on the CPUs.

The Pulsar-VMAF shows 2160p74 when a single kernel runs, and 2160p71 per kernel when dual kernels run in a
local server with Intel® Core™ i9-9900K @ 3.6GHz, 64G(DDR4 16GB PC4-21300 x 4), CentOS Linux Release
7.8.2003.

EC2 F1 Instance details (https://aws.amazon.com/ec2/instance-types/f1/)
<br/>
<img src="images/aws_f1_instances.png" width="70%">
<br/>


# 4 REST API
Once created a F1 instance with Pulsa-VMAF AMI in the AWS Marketplace, the EC2 instance works as an application server.

### Executing Pulsar-VMAF

```bash
curl http://<ip-address>/vmaf -X POST \
-H 'Content-Type: application/json' \
-d '{ \
"ref": <URL of the reference video file>, \
"dst": <URL of the distorted video file>, \
"frames": <number_of_frames>, \
"aws_access_key_id": <AWS_ACCESS_KEY_ID>, \
"aws_secret_access_key": <AWS_SECRET_ACCESS_KEY>}'

-- response --
{
  "task_id": "9ffc50e2-5197-34bb-aaf6-cd1da79f591b", 
  "task_result_url": "http://<ip-address>/9ffc50e2-5197-34bb-aaf6-cd1da79f591b"
}
```

<b>Parameters</b>
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ref</td>
      <td>[Required] URL of Reference(original video) file. Supported protocols are http, https, s3, file</td>
    </tr>
    <tr>
      <td>dst</td>
      <td>[Required] URL of Distorted video from tde original video. Supported protocls are http, https, s3, file</td>
    </tr>
    <tr>
      <td>frames</td>
      <td>[Optional] The number of frames to be processed.</td>
    </tr>
    <tr>
      <td>aws_access_key_id</td>
      <td>AWS access key id from AWS IAM. It is REQUIRED where reference and distorted video are stored in AWS S3.</td>
    </tr>
    <tr>
      <td>aws_secret_access_key</td>
      <td>AWS secret access key from AWS IAM. It is REQUIRED where reference and distorted video are stored in AWS S3.</td>
    </tr>
  </tbody>
</table>

<b>Returns</b>
<table>
  <thead>
    <tr>
      <th>Field</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>task_id</td>
      <td>Unique id of the task. You can get the status and the result of the task with the id.</td>
    </tr>
    <tr>
      <td>task_result_url</td>
      <td>You can monitor the status of the task and get the result from the task_info_url.</td>
    </tr>
  </tbody>
</table>

### Getting status of the task and its results.
```bash
curl http://<ip-address>/<task_id> -X GET \
-H 'Accept: application/json' 
```

You'll receive the following result while the task is running.
```bash
{
  "status": "ready" or "running" 
}
```

When the task is done, you can get the result of the task as the following:
```bash
{
  "status": "done",
  "vmaf": {
    "min": 56.664034,
    "max": 100.000000,
    "mean": 83.260948
  },
  "log_url": "http://<ip-address>/<task_id>/result.json"
}
```
<b>Returns</b>
<table>
  <thead>
    <tr>
      <th>Field</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>status</td>
      <td>status of the task. "ready", "running" or "done"</td>
    </tr>
    <tr>
      <td>vmaf</td>
      <td>The summary of the result when the value of the status is "done".</td>
    </tr>
    <tr>
      <td>log_url</td>
      <td>URL of the log file which has more informations of the result.</td>
    </tr>
  </tbody>
</table>


### Downloading the log file
```bash
curl -o result.log http://<ip-address>/<task_id>/result.json

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  109k  100  109k    0     0  69929      0  0:00:01  0:00:01 --:--:-- 69886
```
# 5 CHANGELOG

## 5.1 libbdvmaf
[0-9.10] 
- Allocate a core automatically.
## 5.2 ffmpeg-bluedot
[1-0.9]
- Removed coreno option of libbdvmaf filter.<br/>
[2.0.0]
- renewal REST API


# 6 CONTACT US

For evaluation of the product, please visit https://blue-dot.io/Evaluation.
For technical problems and questions, feel free to email us at <info@blue-dot.io>
