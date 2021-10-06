**Table Of Contents**

[1 INTRODUCTION](#1-introduction)<br/>
&emsp;[1.1 SPECIFICATION](#11-specification)<br/> 
&emsp;[1.2 PERFORMANCE FOR VARIOUS RESOLUTIONS](#12-performance-for-various-resolutions)<br/>
&emsp;[1.3 BENCHMARK](#13-benchmark)<br/>
[2 EVALUATION](#2-evaluation)<br/>
&emsp;[2.1 AWS F1 INSTANCE](#21-aws-f1-instance)<br/>
&emsp;&emsp;[2.1.1 INSTALLATION](#211-installation)<br/>
&emsp;&emsp;[2.1.2 HOW TO EVALUATE](#212-how-to-evaluate)<br/>
&emsp;[2.2 ALVEO U50](#22-alveo-u50)<br/>
&emsp;&emsp;[2.2.1 GETTING A CREDENTIAL FILE](#221-getting-a-credential-file)<br/>
&emsp;&emsp;[2.2.2 INSTALLATION](#222-installation)<br/>
&emsp;&emsp;[2.2.3 HOW TO EVALUATE](#223-how-to-evaluate)<br/>
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
ssh -i “your_keypair.pem” centos@ec2-12-345-678-901.compute-1.amazonaws.com
```
You should replace the your_keypair.pem and the ce2-12-345-678-901.copmpute-1.amazonaws.com with yours.
If it’s correctly created an instance and connected to it, you can see the following messages:
```
| __| _ \/ __| /_\   |   \| __\ \ / /   /_\ |  \/  |_ _|
| _||  _/ (_ |/ _ \  | |) | _| \ V /   / _ \| |\/| || |
|_| |_|  \___/_/ \_\ |___/|___| \_/   /_/ \_\_|  |_|___|
AMI Version:        1.10.0
Xilinx Version:     2020.2
Readme:             /home/centos/src/README.md
AMI Release Notes:  /home/centos/src/RELEASE_NOTES.md
GUI/Cluster setup:  https://github.com/aws/aws-fpga/blob/master/developer_resources
Developer Support:  https://github.com/aws/aws-fpga/blob/master/README.md#developer-support
Centos Common code: /srv/git/centos-git-common
Xilinx XRT source:  https://github.com/Xilinx/XRT
```
The list of files is as follows at the initial connection:
- 2160_dst.mp4 - distorted video of which quality is to be measured, playback time: 6sec @ 25fps
- 2160.mp4 - reference video
- vmaf_4k_v0.6.1.json, vmaf_v0.6.1.json - VMAF models
    + Refer to [https://github.com/Netflix/vmaf/blob/master/resource/doc/models.md](https://github.com/Netflix/vmaf/blob/master/resource/doc/models.md) for more details.
- f1_binary.xclbin - AWS FPGA binary

##### Step 5. Update software package
Install the latest software package using the following commands:
```bash
sudo yum install libbdvmaf-aws
sudo yum install ffmpeg-bluedot
```
It’s now ready to evaluate the accelerator.
### 2.1.2 HOW TO EVALUATE
In one f1.2xlarge instance, two kernels are instantiated. The following examples show how to run one kernel and two kernels, respectively.  
In the AMI installed, two video bistreams, 2160.mp4 and 2160_dst.mp4, are provided for examples.
#### Measuring VMAF of compressed videos using one kernel
```bash
ffmpeg -stream_loop 99 -i 2160_dst.mp4 -vsync 0 -stream_loop 99 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin -f null -
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
ffmpeg -stream_loop 99 -i 2160_dst.mp4 -vsync 0 -stream_loop 99 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin:log_path=log.json:log_fmt=json -f null -
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
ffmpeg -I 2160_dst.mp4 2160_dst.yuv
```
Run the following command to measure VMAF score of those raw videos.   
```bash
ffmpeg -stream_loop 99 -pix_fmt yuv420p -s 3840x2160 -i 2160_dst.yuv -stream_loop 99 -s 3840x2160 -pix_fmt yuv420p -i 2160.yuv -lavfi libbdvmaf=model_path=vmaf_v0.6.1.json:kernel_path=f1_binary.xclbin:shortest=1 -f null -
```
The same score as the case of compressed videos is reported, and the speed is a little bit better due to less computation load on the CPUs.
#### Running two kernels
```bash
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin -f null -
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=vmaf_4k_v0.6.1.json:kernel_path=f1_binary.xclbin -f null -
```
## 2.2 ALVEO U50
Pulsar-VMAF supports on-premis environment using Xilinx Alveo U50 card.
For more information about Xilinx Alveo U50 Accelator card, please visit the Xilinx website.
[https://www.xilinx.com/products/boards-and-kits/alveo/u50.html](https://www.xilinx.com/products/boards-and-kits/alveo/u50.html)

### 2.2.1 GETTING A CREDENTIAL FILE
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



### 2.2.2 INSTALLATION

---
**Note**  
CentOS 7 is **HIGHLY** recommended for your easy installation.  
We are preparing the software packages for other linux distributions.

---

- Install XRT runtime library from (XRT Github)(https://github.com/Xilinx/XRT)
- Install the latest software package using the following commands:
```bash
sudo yum-config-manager --add-repo http://bluedot-yum-repo.s3-website-us-east-1.amazonaws.com/bluedot-yum.repo
sudo yum install libbdvmaf-u50
sudo yum install ffmpeg-bluedot
```
> :information_source: **Your system may require more rpm packages such as libXv, libav, etc for ffmpeg**

### 2.2.3 HOW TO EVALUATE
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
cp /etc/bluedot/libbdvmaf/conf.json .
cp <your_download_folder>/cred.json .
bddrm.exe /etc/bluedot/libbdvmaf/u50_binary.xclbin
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
ffmpeg -i <transcoded_video_path> -vsync 0 -i <original_video_path> -vsync 0 -lavfi libbdvmaf=model_path=/etc/bluedot/libbdvmaf/vmaf_4k_v0.6.1.json:kernel_path=/etc/bluedot/libbdvmaf/u50_binary.xclbin -f null -
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
ffmpeg -pix_fmt yuv420p -s 3840x2160 -i distorted.yuv -s 3840x2160 -pix_fmt yuv420p -i reference.yuv -lavfi libbdvmaf=model_path=/etc/bluedot/libbdvmaf/vmaf_v0.6.1.json:kernel_path=/etc/bluedot/libbdvmaf/u50_binary.xclbin:shortest=1 -f null -
```
The same score as the case of compressed videos is reported, and the speed is a little bit better due to less computation load on the CPUs.
#### Running two kernels
```bash
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=/etc/bluedot/libbdvmaf/vmaf_4k_v0.6.1.json:kernel_path=/etc/bluedot/libbdvmaf/etc/blutdoeedot/libbdvmaf/u50_binary.xclbin -f null -
ffmpeg -i 2160_dst.mp4 -vsync 0 -i 2160.mp4 -vsync 0 -lavfi libbdvmaf=model_path=/etc/bluedot/libbdvmaf/vmaf_4k_v0.6.1.json:kernel_path=/etc/bluedot/libbdvmaf/etc/blutdoeedot/libbdvmaf/u50_binary.xclbin -f null -
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
Pulsar-VMAF provides REST API.

--- 
**Note** 
> Current version of the REST API is alpha release. We're working on adding more useful functionalities and making it stable.

---

## 4.1 HOWTO
### Creating an account
You can create your account using the REST API.
```bash
curl https://api.kokoon.cloud/auth/signup -X POST \
-H 'Content-Type: application/json' \
-d '{"email": "<email address>", "password": "<password>", "nickname": "<nickname>"}'

-- response --

"" <-- success message
```

You'll get empty message with the status code of 200 if no error.
To verify your email address, we send an email to your email address with a verification code.
Once you receive the verification code, send us the code using the REST API as the following to activate your account
```bash
curl https://api.kokoon.cloud/auth/signup/confirm -X POST \
-H 'Content-Type: application/json' \
-d '{"email": "<email address>", "code": "<confirmation code>"}'

-- response --

"" <-- success message
```
It is success if the API returns empty message.


### Login
You need to get a JWT key to access the REST APIs. You can simply get the JWT key as the following:
```bash
curl https://api.kokoon.cloud/auth -X POST --user '<email address>:<your password>'

-- response --
{"id": "17xxxxd2-3xx4-4xx0-8xx4-3xxxxxxxxxx2", "token": "eyJraWQiO...-yMkzfA4DTnNOfZ0Og", "exp": 3600}
```

The API returns your id, JWT token and expiration time in second as the above.
You should include the token for every HTTP requests. 

### Executing Pulsar-VMAF
The Pulsar-VMAF REST API requires two parameters: one is reference video, the other is distorted video.
Each video can be accessible through HTTP protocol.

```bash
curl https://api.kokoon.cloud/vmaf -X POST \
-H 'Content-Type: application/json' \
-H 'Authorization: bearer eeyJra.eyJzdWIiOiIxN2Y0OGRkMi0zMjk0LTQ4ZTAtO...-yMkzfA4DTnNOfZ0Og' \
-d '{"ref": "https://s3.amazonaws.com/mytest/reference.mp4", "dst": "https://s3.amazonaws.com/mytest/distortion.mp4"}'

-- response --
{"task_id": "9ffc50e2-5197-34bb-aaf6-cd1da79f591b", "update_period": 5, "progress": "in-queue"}
```

Once the task is finished, you'll receive an email as following:
```
Hello,

Your job has been done.



Task ID: 9ffc50e2-5197-34bb-aaf6-cd1da79f591b
Reference video: https://s3.amazonaws.com/mytest/reference.mp4
Distorted video: https://s3.amazonaws.com/mytest/distortion.mp4

VMAF Score: 75.637862
LOG URL   : /fileurl/vmaf/9f971f82-0ecb-11ec-b399-0e1904df6487
```

### Downloading the log file 
You can get download url by the following command line.
```bash
curl https://api.kokoon.cloud/fileurl/vmaf/9f971f82-0ecb-11ec-b399-0e1904df6487 -X POST \
-H 'Content-Type: application/json' \
-H 'Authorization: bearer eeyJraWQiOiJxTHJoRVp5bTFDODF0NkJvVlVxVExxU2s2QUJTWGVWTjRQbU1LcW9aSEF
FPSIsImFsZyI6IlJTMjU2In0.eyJzdWIiOiIxN2Y0OGRkMi0zMjk0LTQ4ZTAtO...-yMkzfA4DTnNOfZ0Og' 

-- response --
{"download_url": "https://startrekapp152952-prod.s3.amazonaws.com/private/us-east-1%3A08920b4c
-885d-4173-86f9-f85f85305b90/vmaf/log/238da3ac-0e99-11ec-879e-0ef222c9480d?AWSAccessKeyId=AS...
CBB75&Signature=7ihve2PPG%2BDUwGlj4qDw8sjXpTY%3D&x-amz-security-token=FwoGZXIvYXdzEDAaDNMKmxB...
coiYbViQYyKJ57Hpw88vT%2BOV3rhMiurzbxmx%2Fprp6tirQcSgL4qOfkSDh0%2B1Nw6Bc%3D&Expires=1630885114"}
```

Now you can download log file from "download_url" of the above result.

```bash
curl -o result.log https://startrekapp152952-prod.s3.amazonaws.com/private/us-east-1%3A08920b4c-
885d-4173-86f9-f85f85305b90/vmaf/log/238da3ac-0e99-11ec-879e-0ef222c9480d?AWSAccessKeyId=AS...CB
B75&Signature=7ihve2PPG%2BDUwGlj4qDw8sjXpTY%3D&x-amz-security-token=FwoGZXIvYXdzEDAaDNMKmxB...co
iYbViQYyKJ57Hpw88vT%2BOV3rhMiurzbxmx%2Fprp6tirQcSgL4qOfkSDh0%2B1Nw6Bc%3D&Expires=1630885114

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


# 6 CONTACT US

For evaluation of the product, please visit https://blue-dot.io/Evaluation.
For technical problems and questions, feel free to email us at <info@blue-dot.io>
