![WaveFrag Logo](docs/images/WF_Rev001.png)

[![GitHub release](https://img.shields.io/github/v/release/wavefrag/WaveFrag-Acoustic-Camera)](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/releases)
[![Actions Status](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/actions/workflows/tests.yml/badge.svg)](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/actions)
[![License](https://img.shields.io/github/license/wavefrag/WaveFrag-Acoustic-Camera)](LICENSE)

# WaveFrag-AcousticCamera
WaveFrag 声学相机提供高性能声学成像与信号处理系统，适用于实验室和个人开发者进行多通道声源定位与分析。  
WaveFrag Acoustic Camera provides a high-performance acoustic imaging and signal processing system, suitable for labs and individual developers for multi-channel source localization and analysis.

## Wiki 文档 / Wiki Documentation
项目文档在 GitHub Wiki 中维护，方便查看和持续更新：  
The project documentation is maintained on GitHub Wiki for easy access and continuous updates:
- 📖 [Home](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Home)
- 🚀 [Getting Started](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Getting_Started)
- 🛠 [Hardware](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Hardware)
- 💻 [Software](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Software)
- ⚠️ [Troubleshooting](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Troubleshooting)
- 📑 [API Reference (optional)](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/API_Reference)

或者您想要下载PDF说明书？
[请点击此链接下载](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/blob/main/docs/WaveFrag_UserManual_v1.0.pdf)

## Hardware Features / 硬件功能
- 多通道麦克风阵列采集 / Multi-channel microphone array acquisition
- 可配置网络参数 / Configurable network parameters
- 即插即用 / Plug and Play
- 可配置硬件滤波器参数 / Configurable hardware filter parameters

## Software Features / 软件功能
- 多通道麦克风阵列采集 / Multi-channel microphone array acquisition
- 实时声源定位与声压图生成 / Real-time source localization and SPL map generation
- 数据可视化功能 / Data visualization
- 支持 FFT、Beamforming 等信号处理算法 / Support FFT, Beamforming and other signal processing algorithms

## Installation / 安装
### GUI 软件 / GUI Software
- Windows / Linux / MacOS 可执行程序包 / Executable package for Windows, Linux, MacOS
- 安装指南详见 [Getting Started](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Getting_Started) / Installation guide see [Getting Started](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Getting_Started)

## Example / 示例
以下示例展示如何使用 Matlab 配合本设备进行多通道声源示波：  
The following example demonstrates how to use Matlab in conjunction with this device to perform multi-channel sound source visualization:

```matlab
% 作者：WaveFrag
%功能：多通道时域、频域信号示波
clc;
clear all;

%% UDP 初始化
feature('DefaultCharacterSet','UTF8');
u = udpport('IPV4','LocalHost','192.168.0.82','LocalPort',3860);
u.Timeout = 0.05;
u.ByteOrder = 'little-endian';

%% 参数设置
FreSampling = 80000;         % 采样率
FramePerSecond = 100;        % 刷新帧率
num_cols = 128;              % 通道数
FrameLen = round((2/FramePerSecond) * FreSampling);  % 单帧长度
buffer_size = FrameLen;

%% 通道选择（重排128通道为16通道）
row_indices = zeros(1,128);
for k = 1:8
    start_idx = k;
    end_idx = 120 + k;
    indices = start_idx:8:end_idx;
    row_indices((k-1)*16+1:k*16) = indices;
end
num_signals = length(row_indices);

%% 初始化绘图界面
fig = figure('Name','UDP 实时示波器','NumberTitle','off','Position',[100 100 950 600]);

% 通道选择下拉框
uicontrol('Style','text','String','显示通道：',...
    'Position',[40 560 80 20],'HorizontalAlignment','left','FontSize',9);
popup = uicontrol('Style','popupmenu',...
    'String',arrayfun(@(x) sprintf('通道 %d',x),1:num_signals,'UniformOutput',false),...
    'Position',[120 560 100 22],'FontSize',9);

% 开始、停止按钮
btnStart = uicontrol('Style','pushbutton','String','开始示波',...
    'Position',[250 558 100 25],'FontSize',9,'BackgroundColor',[0.7 1 0.7]);
btnStop = uicontrol('Style','pushbutton','String','停止示波',...
    'Position',[360 558 100 25],'FontSize',9,'BackgroundColor',[1 0.7 0.7]);

% ==== 预分配 ====
signal = zeros(num_signals, buffer_size, 'double');
data_dec = zeros(1, FrameLen*num_cols, 'int16');
data_matrix = zeros(num_cols, FrameLen, 'int16');
data_matrix_reordered = zeros(num_signals, FrameLen, 'int16');
sig = zeros(1, buffer_size, 'double');
X = zeros(1, buffer_size/2, 'double');
tmp_fft = zeros(1, buffer_size, 'double');

% ==== 绘图初始化 ====
ax1 = subplot(2,1,1);
hPlotTime = plot(ax1, 1:buffer_size, zeros(1,buffer_size));
title(ax1,'实时波形');
xlabel(ax1,'采样点');
ylabel(ax1,'幅度');

ax2 = subplot(2,1,2);
fAxis = linspace(0, FreSampling/2, buffer_size/2);
hPlotFreq = plot(ax2, fAxis, zeros(1,buffer_size/2));
title(ax2,'实时频谱');
xlabel(ax2,'频率 (Hz)');
ylabel(ax2,'幅度');
xlim(ax2,[100 FreSampling/2]);

%% 状态变量
isRunning = false;  % 控制采集循环

% 按钮回调函数
btnStart.Callback = @(~,~) assignin('base','isRunning',true);
btnStop.Callback = @(~,~) assignin('base','isRunning',false);

%% 主循环
while ishandle(fig)
    % 检查运行状态
    if evalin('base','isRunning')
        % 读取UDP数据
        raw = read(u, FrameLen*num_cols, 'int16');
        if isempty(raw)
            continue;
        end
        
        % 使用预分配内存
        data_dec(:) = raw;
        num_data = numel(data_dec);
        num_rows = num_data / num_cols;
        data_matrix(:) = reshape(data_dec, num_cols, num_rows);
        data_matrix_reordered(:) = data_matrix(row_indices, :);
        
        % 当前通道选择
        ch = popup.Value;
        sig(:) = double(data_matrix_reordered(ch, :));
        
        % 更新时域
        set(hPlotTime, 'YData', sig);
        
        % 更新频域
        tmp_fft(:) = abs(fft(sig, buffer_size));
        X(:) = tmp_fft(1:buffer_size/2);
        set(hPlotFreq, 'YData', X);
        
        % 显示
        drawnow limitrate;
        flush(u);
    else
        pause(0.05); % 停止状态下小睡一下，防止空转CPU
    end
end
% 添加示例 Matlab 代码 / Add example Matlab code here
```

![Example Result](docs/images/example_result.png)


## Troubleshooting & FAQ / 常见问题
- 常见问题与解决方法请参见 [Troubleshooting](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Troubleshooting)  
For common issues and solutions, please refer to [Troubleshooting](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/wiki/Troubleshooting)

## License / 许可
WaveFrag-Acoustic-Camera is licensed under MIT License. See [LICENSE](LICENSE)

## Contact / Support / 联系方式
- 技术支持邮箱 / Support Email: support@wavefrag.com
- 论坛与讨论 / Forum and Discussions: [GitHub Discussions](https://github.com/wavefrag/WaveFrag-Acoustic-Camera/discussions)
