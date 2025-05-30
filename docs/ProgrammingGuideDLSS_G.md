Streamline - DLSS-G
=======================

NVIDIA DLSS Frame Generation (“DLSS-FG” or “DLSS-G”) is an AI based technology that infers frames based on rendered frames coming from a game engine or rendering pipeline. This document explains how to integrate DLSS-G into a renderer.

Version 2.7.32
=======

### 0.0 Integration checklist

See Section 15.0 for further details on some of these items, in addition to the Sections noted in the table below.

Item | Reference | Confirmed
---|---|---
All the required inputs are passed to Streamline: depth buffers, motion vectors, HUD-less color buffers  | [Section 5.0](#50-tag-all-required-resources) |
Common constants and frame index are provided for **each frame** using slSetConstants and slSetFeatureConstants methods   |  [Section 7.0](#70-provide-common-constants) |
All tagged buffers are valid at frame present time, and they are not re-used for other purposes | [Section 5.0](#50-tag-all-required-resources) |
Buffers to be tagged with unique id 0 | [Section 5.0](#50-tag-all-required-resources) |
Make sure that frame index provided with the common constants is matching the presented frame | [Section 8.0](#80-integrate-sl-reflex) |
Inputs are passed into Streamline look correct, as well as camera matrices and dynamic objects | [SL ImGUI guide](<Debugging - SL ImGUI (Realtime Data Inspection).md>) |
Application checks the signature of sl.interposer.dll to make sure it is a genuine NVIDIA library | [Streamline programming guide, section 2.1.1](./ProgrammingGuide.md#211-security) |
Requirements for Dynamic Resolution are met (if the game supports Dynamic Resolution)  | [Section 10.0](#100-dlss-g-and-dynamic-resolution) |
DLSS-G is turned off (by setting `sl::DLSSGOptions::mode` to `sl::DLSSGMode::eOff`) when the game is paused, loading, in menu and in general NOT rendering game frames and also when modifying resolution & full-screen vs windowed mode | [Section 12.0](#120-dlss-g-and-dxgi) |
Swap chain is recreated every time DLSS-G is turned on or off (by changing `sl::DLSSGOptions::mode`) to avoid unnecessary performance overhead when DLSS-G is switched off | [Section 18.0](#180-how-to-avoid-unnecessary-overhead-when-dlss-g-is-turned-off) |
Reduce the amount of motion blur; when DLSS-G enabled, halve the distance/magnitude of motion blur | N/A |
Reflex is properly integrated (see checklist in Reflex Programming Guide) | [Section 8.0](#80-integrate-sl-reflex) |
In-game UI for enabling/disabling DLSS-G is implemented | [RTX UI Guidelines](<RTX UI Developer Guidelines.pdf>) |
Only full production non-watermarked libraries are packaged in the release build | N/A |
No errors or unexpected warnings in Streamline and DLSS-G log files while running the feature | N/A |
Ensure extent resolution or resource size, whichever is in use, for `Hudless` and `UI Color and Alpha` buffers exactly match that of backbuffer. | N/A |

### 1.0 REQUIREMENTS

**NOTE - DLSS-G requires the following Windows versions/settings to run.  The DLSS-G feature will fail to be available if these are not met.  Failing any of these will cause DLSS-G to be unavailable, and Streamline will log an error:**

* Minimum Windows OS version of Win10 20H1 (version 2004, build 19041 or higher)
* Display Hardware-accelerated GPU Scheduling (HWS) must be enabled via Settings : System : Display : Graphics : Change default graphics settings.

#### 1.1 Run-time Performance

DLSS-G's execution time and memory footprint vary across different engines and
integrations. To provide a general guideline, NVIDIA has profiled DLSS-G in-game
on various NVIDIA GeForce RTX GPUs. These results serve as a rough estimate of
run-time execution and can help developers estimate the potential savings
offered by DLSS.

**Execution Cost**

| GeForce SKU | Resolution | 2x Cost (ms) | 4x Cost (ms) |
|-------------|------------|--------------|--------------|
| RTX 4090    | 1080p      | 1.35         | -            |
| RTX 4090    | 1440p      | 1.78         | -            |
| RTX 4090    | 4k         | 2.77         | -            |
| RTX 5080    | 1080p      | 1.45         | 2.43         |
| RTX 5080    | 1440p      | 2.04         | 3.61         |
| RTX 5080    | 4k         | 2.84         | 5.25         |
| RTX 5090    | 1080p      | 1.07         | 1.78         |
| RTX 5090    | 1440p      | 1.46         | 2.48         |
| RTX 5090    | 4k         | 1.72         | 3.32         |

**Memory Cost**

The DLSS-G memory footprint does not change significantly between single-frame
generation and multi-frame generation modes.

| Resolution | VRAM Estimate (MB) |
|------------|--------------------|
| 1080p      | 272                |
| 1440p      | 489                |
| 4k         | 725                |

### 2.0 INITIALIZATION AND SHUTDOWN

Call `slInit` as early as possible (before any d3d12/vk APIs are invoked)

```cpp
#include <sl.h>
#include <sl_consts.h>
#include <sl_dlss_g.h>

sl::Preferences pref;
pref.showConsole = true; // for debugging, set to false in production
pref.logLevel = sl::eLogLevelDefault;
pref.pathsToPlugins = {}; // change this if Streamline plugins are not located next to the executable
pref.numPathsToPlugins = 0; // change this if Streamline plugins are not located next to the executable
pref.pathToLogsAndData = {}; // change this to enable logging to a file
pref.logMessageCallback = myLogMessageCallback; // highly recommended to track warning/error messages in your callback
pref.applicationId = myId; // Provided by NVDA, required if using NGX components (DLSS 2/3)
pref.engineType = myEngine; // If using UE or Unity
pref.engineVersion = myEngineVersion; // Optional version
pref.projectId = myProjectId; // Optional project id
if(SL_FAILED(res, slInit(pref)))
{
    // Handle error, check the logs
    if(res == sl::Result::eErrorDriverOutOfDate) { /* inform user */}
    // and so on ...
}
```

For more details please see [preferences](ProgrammingGuide.md#222-preferences)

Call `slShutdown()` before destroying dxgi/d3d12/vk instances, devices and other components in your engine.

```cpp
if(SL_FAILED(res, slShutdown()))
{
    // Handle error, check the logs
}
```

#### 2.1 SET THE CORRECT DEVICE

Once the main device is created call `slSetD3DDevice` or `slSetVulkanInfo`:

```cpp
if(SL_FAILED(res, slSetD3DDevice(nativeD3DDevice)))
{
    // Handle error, check the logs
}
```

### 3.0 CHECK IF DLSS-G IS SUPPORTED

As soon as SL is initialized, you can check if DLSS-G is available for the specific adapter you want to use:

```cpp
Microsoft::WRL::ComPtr<IDXGIFactory> factory;
if (SUCCEEDED(CreateDXGIFactory(__uuidof(IDXGIFactory), (void**)&factory)))
{
    Microsoft::WRL::ComPtr<IDXGIAdapter> adapter{};
    uint32_t i = 0;
    while (factory->EnumAdapters(i, &adapter) != DXGI_ERROR_NOT_FOUND)
    {
        DXGI_ADAPTER_DESC desc{};
        if (SUCCEEDED(adapter->GetDesc(&desc)))
        {
            sl::AdapterInfo adapterInfo{};
            adapterInfo.deviceLUID = (uint8_t*)&desc.AdapterLuid;
            adapterInfo.deviceLUIDSizeInBytes = sizeof(LUID);
            if (SL_FAILED(result, slIsFeatureSupported(sl::kFeatureDLSS_G, adapterInfo)))
            {
                // Requested feature is not supported on the system, fallback to the default method
                switch (result)
                {
                    case sl::Result::eErrorOSOutOfDate:         // inform user to update OS
                    case sl::Result::eErrorDriverOutOfDate:     // inform user to update driver
                    case sl::Result::eErrorNoSupportedAdapter:  // cannot use this adapter (older or non-NVDA GPU etc)
                    // and so on ...
                };
            }
            else
            {
                // Feature is supported on this adapter!
            }
        }
        i++;
    }
}
```

#### 3.1 CHECKING DLSS-G'S CONFIGURATION AND SPECIAL REQUIREMENTS

In order for DLSS-G to work correctly certain requirements regarding the OS, driver and other settings on user's machine must be met. To obtain DLSS-G configuration and check if all requirements are met you can use the following code snippet:

```cpp
sl::FeatureRequirements requirements{};
if (SL_FAILED(result, slGetFeatureRequirements(sl::kFeatureDLSS_G, requirements)))
{
    // Feature is not requested on slInit or failed to load, check logs, handle error
}
else
{
    // Feature is loaded, we can check the requirements    
    requirements.flags & FeatureRequirementFlags::eD3D12Supported
    requirements.flags & FeatureRequirementFlags::eVulkanSupported
    requirements.maxNumViewports
    // and so on ...
}
```
> **NOTE:**
> DLSS-G runs optical flow in interop mode in Vulkan by default. In order to leverage potential performance benefit of running optical flow natively in Vulkan, client must meet the minimum requirements of Nvidia driver version being 527.64 on Windows and 525.72 on Linux and VK_API_VERSION_1_1 (recommended version - VK_API_VERSION_1_3).
> In manual hooking mode, it must meet additional requirements as described in section 5.2.1 of ProgrammingGuideManualHooking.md.

### 4.0 HANDLE MULTIPLE SWAP-CHAINS

DLSS-G will automatically attach to any swap-chain created by the application **unless manual hooking is used**. In the editor mode there could be multiple swap-chains but DLSS-G should attach only to the main one where frame interpolation is used.
Here is how DLSS-G could be enabled only on a single swap-chain:

```cpp
// This is just one example, swap-chains can be created at any point in time and in any order.
// SL features also can be loaded/unloaded at any point in time and in any order.

// Unload DLSS-G (this can be done at any point in time and as many times as needed)
slSetFeatureLoaded(sl::kFeatureDLSS_G, false);

// Create swap chains for which DLSS-G is NOT required
IDXGISwapChain1* swapChain{};
factory->CreateSwapChainForHwnd(device, hWnd, desc, nullptr, nullptr, &swapChain);
// and so on

// Load DLSS-G (this can be done at any point in time and as many times as needed)
slSetFeatureLoaded(sl::kFeatureDLSS_G, true);

// Create main swap chains for which DLSS-G is required
IDXGISwapChain1* mainSwapChain{};
factory->CreateSwapChainForHwnd(device, hWnd, desc, nullptr, nullptr, &mainSwapChain);

// From this point onward DLSS-G will automatically manage only mainSwapChain, other swap-chains use standard DXGI implementation

```

### 5.0 TAG ALL REQUIRED RESOURCES

#### **Buffers to tag**

DLSS-G requires `depth` and `motion vectors` buffers.

If DLSS-G needs to run only on a subregion of the final color buffer, hereafter referred to as backbuffer subrect, then it is required to tag the backbuffer, only to pass in backbuffer subrect info while optionally passing in backbuffer resource pointer. Refer to [Tagging Recommendations section](#tagging-recommendations) below for details.

Additionally, for maximal image quality, it is **critical** to integrate `UI Color and Alpha` or `Hudless` buffers:
* `UI Color and Alpha` buffer provides significant image quality improvements on UI elements like name plates and on-screen hud. If your application/game has this available, we strongly recommend you integrate this buffer.
* If `UI Color and Alpha` is not available, `Hudless` integration can also significantly improve image quality on UI elements.
* Extent resolution or resource size, whichever is in use, for `Hudless` and `UI Color and Alpha` buffers should exactly match that of backbuffer.

Input | Requirements/Recommendations | Reference Image
---|---|---
Final Color | - *No requirements, this is intercepted automatically via SL's SwapChain API* | ![dlssg_final_color](./media/dlssg_docs_final_color.png "DLSSG Input Example: Final Color")
Final Color Subrect | - Subregion of the final color buffer to run frame-generation on. <br> - Subrect-external backbuffer region is copied as is to the generated frame. <br> - Tag backbuffer optionally, only to pass in backbuffer subrect info. <br> - Extent resolution or resource size, whichever is in use, for `Hudless` and `UI Color and Alpha` buffers should exactly match that of backbuffer. <br> - Refer to [Tagging Recommendations section](#tagging-recommendations) below for details. | ![dlssg_final_color_subrect](./media/dlssg_docs_final_color_subrect.png "DLSSG Input Example: Final Color Subrect")
Depth | - Same depth data used to generate motion vector data <br> - `sl::Constants` depth-related data (e.g. `depthInverted`) should be set accordingly<br>  - *Note: this is the same set of requirements as DLSS-SR, and the same depth can be used for both* | ![dlssg_depth](./media/dlssg_docs_depth.png "DLSSG Input Example: Depth")
Motion Vectors | - Dense motion vector field (i.e. includes camera motion, and motion of dynamic objects) <br> - *Note: this is the same set of requirements as DLSS-SR, and the same motion vectors can be used for both* | ![dlssg_mvec](./media/dlssg_docs_mvec.png "DLSSG Input Example: Motion Vectors")
Hudless | - Should contain the full viewable scene, **without any HUD/UI elements in it**. If some HUD/UI elements are unavoidably included, expect some image quality degradation on those elements <br> - Same color space and post-processing effects (e.g tonemapping, blur etc.) as color backbuffer <br> - When appropriate buffer extents are *not* provided, needs to have the same dimensions as the color backbuffer <br> | ![dlssg_hudless](./media/dlssg_docs_hudless.png "DLSSG Input Example: Hudless")
UI Color and Alpha | - Should **only** contain pixels that denote the UI/HUD, along with appropriate alpha values (described below) <br> - Alpha is *zero* on all pixels that do *not* have UI on them <br> - Alpha is *non-zero* on all pixels that do have UI on them <br> - RGB is premultiplied by alpha, and is as close as possible to respecting the following blending formula: `Final_Color.RGB = UI.RGB + (1 - UI.Alpha) x Hudless.RGB` <br> - When appropriate buffer extents are *not* provided, needs to have the same dimensions as the color backbuffer <br> | ![dlssg_ui_color_and_alpha](./media/dlssg_docs_ui_color_and_alpha.png "DLSSG Input Example: UI Color and Alpha")
Bidirectional Distortion Field | - Optional buffer, **only needed when strong distortion effects are applied as post-processing filters** <br> - Refer to [pseudo-code below ](#bidirectional-distortion-field-buffer-generation-code-sample) for an example on how to generate this optional buffer <br> - When this buffer is tagged, Mvec and Depth need to be **undistorted** <br> - When this buffer is tagged, the FinalColor is should be **distorted** <br> - When this buffer is tagged, Hudless and UIColorAndAlpha need to be such that `Blend(Hudless, UIColorAndAlpha) = FinalColor`. This may mean that Hudless needs to be equally distorted, and in rare cases that UIColorAndAlpha is also equally distorted <br> - **Resolution**: we recommend using half of the FinalColor's resolution's width and height <br> - **Channel count**: 4 channels <br> - **RG channels**: UV coordinates of the corresponding **undistorted** pixel, as an offset relative to the source UV coordinate <br> - **BA channels**: UV coordinates of the corresponding **distorted** pixel, as an offset relative to the source UV coordinate <br> - **Units**: the buffer values should be in normalized pixel space `[0,1]`. These should be the same scale as the input MVecs <br> - **Channel precision and format:** Signed format, equal bit-count per channel (i.e. R10G10B10A2 is NOT allowed). We recommend a minimum of 8 bits per channel, with precision scale and bias (`PrecisionInfo`) passed in as part of the `ResourceTag` | <center>**Barrel distortion, RGB channels**  ![dlssg_bidirectional_distortion_field](./media/dlssg_docs_bidirectional_distortion_field.png "DLSSG Input Example: Bidirectional Distortion Field") <br><br> <center>**Barrel distortion, absolute value of RG channels** ![dlssg_docs_bidirectional_distortion_field_rg_abs](./media/dlssg_docs_bidirectional_distortion_field_rg_abs.png "DLSSG Input Example: Bidirectional Distortion Field, RG channels, Absolute value") 

#### **Tagging recommendations**

**For all buffers**: tagged buffers are used during the `Swapchain::Present` call. **If the tagged buffers are going to be reused, destroyed or changed in any way before the frame is presented, their life-cycle needs to be specified correctly**.

It is important to emphasize that **the overuse of `sl::ResourceLifecycle::eOnlyValidNow` and `sl::ResourceLifecycle::eValidUntilEvaluate` can result in wasted VRAM**. Therefore please do the following:

* First tag all of the DLSS-G inputs as `sl::ResourceLifecycle::eValidUntilPresent` then test and see if DLSS-G is working correctly.
* Only if you notice that one or more of the inputs (depth, mvec, hud-less, ui etc.) has incorrect content at the `present frame` time, should you proceed and flag them as `sl::ResourceLifecycle::eOnlyValidNow` or `sl::ResourceLifecycle::eValidUntilEvaluate` as appropriate.

In order to run DLSS-G on final color subrect region:
* It is required to tag backbuffer to pass-in subrect data.
* Only buffer type - `kBufferTypeBackbuffer` and backbuffer extent data are required to be passed in when setting the tag for backbuffer; the rest of the other inputs to sl::ResourceTag are optional. This implies passing in NULL backbuffer resource pointer is valid because SL already has knowledge about the backbuffer being presented.
* If a valid backbuffer resource pointer is passed in when tagging:
  * SL will hold a reference to it until a null tag is set.
  * SL will warn if it doesn't match the SL-provided backbuffer resource being presented.

> NOTE:
> SL will hold a reference to all `sl::ResourceLifecycle::eValidUntilPresent` resources until a null tag is set, therefore the application will not crash if host releases tagged resource before `present frame` event is reached. This does not apply to Vulkan.

```cpp

// IMPORTANT: 
//
// Resource state for the immutable resources needs to be correct when tagged resource is used by SL - during the Present call
// Resource state for the volatile resources needs to be correct for the command list used to tag the resource - SL will make a copy which is later on used by DLSS-G during the Present call
// 
// GPU payload that generates content for any volatile resource MUST be either already submitted to the provided command list or some other command list which is guaranteed to be executed BEFORE.

// Prepare resources (assuming d3d12 integration so leaving Vulkan view and device memory as null pointers)
//
// NOTE: As an example we are tagging depth as immutable and mvec as volatile, this needs to be adjusted based on how your engine works
sl::Resource depth = {sl::ResourceType::Tex2d, myDepthBuffer, nullptr, nullptr, depthState, nullptr};
sl::Resource mvec = {sl::ResourceType::Tex2d, myMotionVectorsBuffer, nullptr, mvecState, nullptr, nullptr};
sl::ResourceTag depthTag = sl::ResourceTag {&depth, sl::kBufferTypeDepth, sl::ResourceLifecycle::eValidUntilPresent, &fullExtent }; // valid all the time
sl::ResourceTag mvecTag = sl::ResourceTag {&mvec, sl::kBufferTypeMvec, sl::ResourceLifecycle::eOnlyValidNow, &fullExtent };     // reused for something else later on

// Normally depth and mvec are available at a similar point in the pipeline so tagging them together
// If this is not the case simply tag them separately when they are available
sl::Resource inputs[] = {depthTag, mvecTag};
slSetTagForFrame(*currentFrame, viewport, inputs, _countof(inputs), cmdList);

// Tag backbuffer only to pass in backbuffer subrect info
sl::Extent backBufferSubrectInfo {128, 128, 512, 512}; // backbuffer subrect info to run FG on.
sl::ResourceTag backbufferTag = sl::ResourceTag {nullptr, sl::kBufferTypeBackbuffer, sl::ResourceLifecycle{}, &backBufferSubrectInfo };
sl::Resource inputs[] = {backbufferTag};
slSetTagForFrame(*currentFrame, viewport, inputs, _countof(inputs), cmdList);

// After post-processing pass but before UI/HUD is added tag the hud-less buffer
//
sl::Resource hudLess = {sl::ResourceType::Tex2d, myHUDLessBuffer, nullptr, nullptr, hudlessState, nullptr};
sl::ResourceTag hudLessTag = sl::ResourceTag {&hudLess, sl::kBufferTypeHUDLessColor, sl::ResourceLifecycle::eValidUntilPresent, &fullExtent }; // valid all the time
sl::Resource inputs[] = {hudLessTag};
slSetTagForFrame(*currentFrame, viewport, inputs, _countof(inputs), cmdList);

// UI buffer with color and alpha channel
//
sl::Resource ui = {sl::ResourceType::Tex2d, myUIBuffer, nullptr, nullptr, uiTextureState, nullptr};
sl::ResourceTag uiTag = sl::ResourceTag {&ui, sl::kBufferTypeUIColorAndAlpha, sl::ResourceLifecycle::eValidUntilPresent, &fullExtent }; // valid all the time
sl::Resource inputs[] = {uiTag};
slSetTagForFrame(*currentFrame, viewport, inputs, _countof(inputs), cmdList);

// OPTIONAL! Only need the Bidirectional distortion field when strong distortion effects are applied during post-processing
//
sl::Resource bidirectionalDistortionField = {sl::ResourceType::Tex2d, myBidirectionalDistortionBuffer, nullptr, nullptr, bidirectionalDistortionState};
sl::ResourceTag bidirectionalDistortionTag = sl::ResourceTag {&bidirectionalDistortionField, sl::kBufferTypeBidirectionalDistortionField, sl::ResourceLifecycle::eValidUntilPresent, &fullExtent }; // valid all the time
sl::Resource inputs[] = {bidirectionalDistortionTag};
slSetTagForFrame(*currentFrame, viewport, inputs, _countof(inputs), cmdList);
```

> **NOTE:**
> If dynamic resolution is used then please specify the extent for each tagged resource. Please note that SL **manages resource states so there is no need to transition tagged resources**.

> **IMPORTANT:**
> If validity of tagged resources cannot be guaranteed (for example game is loading, paused, in menu, playing a video cut scene etc.) **all tags should be set to null pointers to avoid stability or IQ issues**.

#### **Multiple viewports**

DLSS-G supports multiple viewports. Resources for each viewport must be tagged independently. Our SL Sample ( https://github.com/NVIDIAGameWorks/Streamline_Sample ) supports multiple viewports. Check the sample for recommended best practices on how to do it. The idea is that resource tags for different resources are independent
from each other. For instance - if you have two viewports, there must be two slSetTagForFrame() calls. Input resource for one viewport may be different from the input resource
for another viewport. However - all viewports do write into the same backbuffer.

Note that DLSS-G doesn't support multiple swap chains at the moment. So all viewports must write into the same backbuffer.

#### **Bidirectional Distortion Field buffer generation code sample**
The following HLSL code snippet demonstrates the generation of the bidirectional distortion field buffer. The example distortion illustrated is barrel distortion.

```cpp
const float distortionAlpha = -0.5f;
 
float2 barrelDistortion(float2 UV)
{    
    // Barrel distortion assumes UVs relative to center (0,0), so we transform
    // to [-1, 1]
    float2 UV11 = (UV * 2.0f) - 1.0f;
     
    // Squared norm of distorted distance to center
    float r2 = UV11.x * UV11.x + UV11.y * UV11.y;
     
    // Reference: http://www.cs.ait.ac.th/~mdailey/papers/Bukhari-RadialDistortion.pdf
    float x = UV11.x / (1.0f + distortionAlpha * r2);
    float y = UV11.y / (1.0f + distortionAlpha * r2);
     
    // Transform back to [0, 1]     
    float2 outUV = float2(x, y);
    return (outUV + 1.0f) / 2.0f;
}
 
float2 inverseBarrelDistortion(float2 UV)
{  
    // Barrel distortion assumes UVs relative to center (0,0), so we transform
    // to [-1, 1]
    float2 UV11 = (UV * 2.0f) - 1.0f;
     
    // Squared norm of undistorted distance to center
    float ru2 = UV11.x * UV11.x +  UV11.y * UV11.y;
 
    // Solve for distorted distance to center, using quadratic formula
    float num = sqrt(1.0f - 4.0f * distortionAlpha * ru2) - 1.0f;
    float denom = 2.0f * distortionAlpha * sqrt(ru2);
    float rd = -num / denom;
     
    // Reference: http://www.cs.ait.ac.th/~mdailey/papers/Bukhari-RadialDistortion.pdf
    float x = UV11.x * (rd / sqrt(ru2));
    float y = UV11.y * (rd / sqrt(ru2));
     
    // Transform back to [0, 1]     
    float2 outUV = float2(x, y);
    return (outUV + 1.0f) / 2.0f;
}

float2 generateBidirectionalDistortionField(Texture2D output, float2 UV)
{
    // Assume UV is in [0, 1]
    float2 rg = barrelDistortion(UV) - UV;
    float2 ba = inverseBarrelDistortion(UV) - UV;
 
    // rg and ba needs to be in the same canonical format as the motion vectors
    // i.e. a displacement of rg or ba needs to to be in the same scale as (Mvec.x, Mvec.y)
     
    // The output can be outside of the [0, 1] range
    Texture2D[UV] = float4(rg, ba); // needs to be signed
}
```

This HLSL code snippet uses an iterative Newton-Raphson method to solve the inverse distortion problem. It is designed to be used directly in shader code, especially when an analytical solution is not available. While the method is effective, it does not guarantee convergence for all distortion functions, so users should verify its suitability for their specific use case.

```cpp
float2 myDistortion(float2 xy)
{
     // The distortion function
}

float loss(float2 Pxy, float2 ab)
{
    float2 Pab = myDistortion(ab);
    float2 delta = Pxy - Pab;
    return dot(delta, delta);
}

float2 iterativeInverseDistortion(float2 UV)
{
    const float kTolerance = 1e-6f;
    const float kGradDelta = 1e-6f;      // The delta used for gradient estimation
    const int kMaxIterations = 5;        // Select a low number of iterations which minimizes the loss
    const int kImprovedInitialGuess = 1; // Assume a locally uniform distortion field 
    
    float2 ab = UV; // initial guess
    
    if (kImprovedInitialGuess)
    {
        ab = UV - (myDistortion(UV) - UV);
    }

    for (int i = 0; i < kMaxIterations; ++i)
    {
        float F = loss(UV, ab);

        // Central difference
        const float Fabx1 = loss(UV, ab + float2(kGradDelta * 0.5f, 0));
        const float Fabx0 = loss(UV, ab - float2(kGradDelta * 0.5f, 0));
        const float Faby1 = loss(UV, ab + float2(0, kGradDelta * 0.5f));
        const float Faby0 = loss(UV, ab - float2(0, kGradDelta * 0.5f));

        float2 grad;
        grad.x = (Fabx1 - Fabx0) / kGradDelta;
        grad.y = (Faby1 - Faby0) / kGradDelta;

        const float norm = grad.x * grad.x + grad.y * grad.y;
        if (abs(norm) < kTolerance) {
            break;
        }

        float delta_x = F * grad.x / norm;
        float delta_y = F * grad.y / norm;

        ab.x = ab.x - delta_x;
        ab.y = ab.y - delta_y;
    }

    return ab;
}
```

### 6.0 SET DLSS-G OPTIONS

slDLSSGSetOptions() is actioned in the following DXGI / VK Present call. As such, it should not be considered thread safe with respect to that Present call. I.e. the application is expected to add any necessary synchronization logic to ensure these all slDLSSGSetOptions() and Present() calls are received by the Streamline in the correct order.

#### 6.1 ENABLING MULTI-FRAME GENERATION

**NOTE: Not all devices support multi-frame generation. To check whether multi-frame generation is supported, use `slDLSSGGetState`. See [Section 15.0](#150-how-to-check-dlss-g-status-at-runtime) for more details.**

By default, the DLSS-G plugin generates a single frame per `Present()` call.
Setting `numFramesToGenerate` to a value greater than one causes DLSS-G to
generate that many frames per `Present()`. For example, if `numFramesToGenerate`
is 3, then Streamline will present four frames each time `Present()` is called:
three generated frames, plus the frame presented by the host.

Note that the value of `numFramesToGenerate` must be between 1 and the maximum
number of generated frames, as reported in
`sl::DLSSGState::numFramesToGenerateMax` (inclusive). Attempting to set a value
outside this range will result in `Result::eErrorInvalidState`.

#### 6.2 TURNING DLSS-G ON/OFF/AUTO

**NOTE: By default DLSS-G interpolation is off, even if the feature is loaded and the required items tagged.  DLSS-G must be explicitly turned on by the application using the DLSS-G-specific constants function.**

DLSS-G options must be set so that the DLSS-G plugin can track any changes made by the user, and to enable DLSS-G interpolation.  To enable interpolation, be sure to set `mode` to `sl::DLSSGMode::eOn` or `sl::DLSSGMode::eAuto` if using [Dynamic Frame Generation](#220-dynamic-frame-generation). While DLSS-G can be turned on/off/auto in development builds via a hotkey, it is best for the application not to rely on this, even during development.

```cpp

// Using helpers from sl_dlss_g.h

sl::DLSSGOptions options{};
// These are populated based on user selection in the UI
options.mode = myUI->getDLSSGMode(); // e.g. sl::DLSSGMode::eOn;
// IMPORTANT: Note that we are using IDENTICAL viewport as when tagging our resources
if(SL_FAILED(result, slDLSSGSetOptions(viewport, options)))
{
    // Handle error here, check the logs
}
```

**When to disable DLSS-G**
- Temporary Events (may retain resources, see below):
    - A fullscreen game menu is entered
    - A translucent UI element is overlaid over the majority of the screen (ex: game leaderboard)
- Persistent Events (must not retain resources):
    - A user has turned off DLSS-G via a settings menu
    - A console command has been used to turn off DLSS-G

#### 6.3 RETAINING RESOURCES WHEN DLSS-G IS OFF
Setting `sl::DLSSGOptions.mode` to `sl::DLSSGMode::eOff` releases all resources
allocated by DLSS-G. These resources will be reallocated when the mode is
changed back to `sl::DLSSGMode::eOn`, which may result in small stutter.

Applications should use the `sl::DLSSGFlags::eRetainResourcesWhenOff` flag to
instruct DLSS-G to not release resources when turned off. Note that to release
DLSS-G resources when this flag is set, `slFreeResources()` must be called. This
must be done whenever DLSS-G is explicitly disabled (for example, via a settings
menu or console command)

**Note:** DLSS-G will continue to automatically allocate/free resources on
events like resolution changes. The `sl::DLSSGFlags::eRetainResourcesWhenOff`
flag has no effect on these implicit events.

#### 6.4 AUTOMATICALLY DISABLING DLSS-G IN MENUS
If `kBufferTypeUIColorAndAlpha` is provided, DLSS-G can automatically detect
fullscreen menus and turn off automatically. To enable automatic fullscreen menu
detection, set the `sl::DLSSGFlags::eEnableFullscreenMenuDetection` flag.
This flag may be changed on a per-frame basis to disable detection on specific
scenes, for example.

Since this approach may not detect menus in all cases, it is still preferred to
disable DLSS-G manually, by setting the mode to `sl::DLSSGMode::eOff`.

**Note:** when DLSS-G is disabled by fullscreen menu detection, its resources
will _always_ be retained, regardless of the value of the
`sl::DLSSGFlags::eRetainResourcesWhenOff` flag

#### 6.5 HOW TO SETUP A CALLBACK TO RECEIVE API ERRORS (OPTIONAL)

DLSS-G intercepts `IDXGISwapChain::Present` and when using Vulkan `vkQueuePresentKHR` and `vkAcquireNextImageKHR`calls and executes them asynchronously. When calling these methods from the host side SL will return the "last known error" but in order to obtain per call API error you must provide an API error callback. Here is how this can be done:

```cpp

// Triggered immediately upon return from the API call but ONLY if return code != 0
void myAPIErrorCallback(const sl::APIError& e)
{
    // Handle error, use e.hres with DirectX and e.vkRes on Vulkan
    
    // IMPORTANT: STORE ERROR AND RETURN IMMEDIATELY TO AVOID STALLING PRESENT THREAD
};

sl::DLSSGOptions options{};
// Constants are populated based on user selection in the UI
options.mode = myUI->getDLSSGMode(); // e.g. sl::eDLSSGModeOn;
options.onErrorCallback = myAPIErrorCallback;
if(SL_FAILED(result, slDLSSGSetOptions(viewport, options)))
{
    // Handle error here, check the logs
}
```

> **NOTE:**
> API error callbacks are triggered from the Present thread and **must not be blocked** for a prolonged period of time.

> **IMPORTANT:**
> THIS IS OPTIONAL AND ONLY NEEDED IF YOU ARE ENCOUNTERING ISSUES AND NEED TO PROCESS SPECIFIC ERRORS RETURNED BY THE VULKAN OR DXGI API

### 7.0 PROVIDE COMMON CONSTANTS

Various per frame camera related constants are required by all Streamline features and must be provided ***if any SL feature is active and as early in the frame as possible***. Please keep in mind the following:

* All SL matrices are row-major and should not contain any jitter offsets
* If motion vector values in your buffer are in {-1,1} range then motion vector scale factor in common constants should be {1,1}
* If motion vector values in your buffer are NOT in {-1,1} range then motion vector scale factor in common constants must be adjusted so that values end up in {-1,1} range

```cpp
sl::Constants consts = {};
// Set motion vector scaling based on your setup
consts.mvecScale = {1,1}; // Values in eMotionVectors are in [-1,1] range
consts.mvecScale = {1.0f / renderWidth,1.0f / renderHeight}; // Values in eMotionVectors are in pixel space
consts.mvecScale = myCustomScaling; // Custom scaling to ensure values end up in [-1,1] range
sl::Constants consts = {};
// Set all constants here
//
// Constants are changing per frame tracking handle must be provided
if(!setConstants(consts, *frameToken, viewport))
{
    // Handle error, check logs
}
```

For more details please see [common constants](ProgrammingGuide.md#2101-common-constants)

### 8.0 INTEGRATE SL REFLEX

**It is required** for sl.reflex to be integrated in the host application. **Please note that any existing regular Reflex SDK integration (not using Streamline) cannot be used by DLSS-G**. Special attention should be paid to the markers `eReflexMarkerPresentStart` and `eReflexMarkerPresentEnd` which must provide correct frame index so that it can be matched to the one provided in the [section 7](#70-provide-common-constants)

For more details please see [Reflex guide](ProgrammingGuideReflex.md)

> **IMPORTANT:**
> If you see a warning in the SL log stating that `common constants cannot be found for frame N` that indicates that sl.reflex markers `eReflexMarkerPresentStart` and `eReflexMarkerPresentEnd` are out of sync with the actual frame being presented.

### 9.0 DLSS-G DEVELOPMENT HOTKEYS

When using non-production (development) builds of `sl.dlss_g.dll`, there are numerous hotkeys available, all of which can be remapped using the remapping methods described in [debugging](<Debugging - JSON Configs (Plugin Configs).md>)

* `"dlssg-sync"` (default `VK_END`)
  * Toggle delaying the presentation of the next frame to experiment with minimizing latency
* `"vsync"` (default `Shift-Ctrl-'1'`)
  * Toggle vsync on output swapchain
* `"debug"` (default `Shift-Ctrl-VK_INSERT`)
  * Toggle debugging view
* `"stats"` (default `Shift-Ctrl-VK_HOME`)
  * Toggle performance stats
* `"dlssg-toggle"` (default `VK_OEM_2` `/?` for US)
  * Toggle DLSS-G on/off/auto (override app setting)
* `"write-stats"` (default `Ctrl-Alt-'O'`)
  * Write performance stats to file

### 10.0 DLSS-G AND DYNAMIC RESOLUTION

DLSS-G supports dynamic resolution of the MVec and Depth buffer extents.  Dynamic resolution may be done via DLSS or an app-specific method.  Since DLSS-G uses the final color buffer with all post-processing complete, the color buffer, or its subrect if in use, must be a fixed size -- it cannot resize per-frame.  When DLSS-G dynamic resolution mode is enabled, the application can pass in a differently-sized extent for the MVec and Depth buffers on a perf frame basis.  This allows the application to dynamically change its rendering load smoothly.

There are a few requirements when using dynamic resolution with DLSS-G:

* The application must set the flag `sl::DLSSGFlags::eDynamicResolutionEnabled` in `sl::DLSSGOptions::flags` when dynamic resolution is active.  It should clear the flag when/if dynamic resolutiuon is disabled.  *DO NOT* leave the dynamic resolution flag set when using fixed-ratio DLSS, as it may decrease performance or image quality.
* The application should specify `sl::DLSSGOptions::dynamicResWidth` and `sl::DLSSGOptions::dynamicResHeight` to a target resolution in the range of the dynamic MVec and Depth buffer sizes.
  * This is the fixed resolution at which DLSS-G will process the MVec and Depth buffers.
  * This value must not change dynamically per-frame.  Changing it outside of the application UI can lead to a frame rate glitch.
  * Set it to a reasonable "middle-range" value and do not change it until/unless the DLSS or other dynamic-range settings change.  
  * For example, if the application has a final, upscaled color resolution of 3840x2160 pixels, with a rendering resolution that can vary between 1920x1080 and 3840x2160 pixels, the `dynamicResWidth` and `Height` could be set to 2880x1620 or 1920x1080.
  * This ratio between the min and max resolutions can be tuned for performance and quality.
  * If the application passes 0 for these values when DLSS-G dynamic resolution is enabled, then DLSS-G will default to half of the resolution of the final color target or its subrect, if in use.

```cpp

// Using helpers from sl_dlss_g.h

sl::DLSSGOptions options{};
// These are populated based on user selection in the UI
options.mode = myUI->getDLSSGMode(); // e.g. sl::eDLSSGModeOn;
options.flags = sl::DLSSGFlags::eDynamicResolutionEnabled;
options.dynamicResWidth = appSelectedInternalWidth;
options.dynamicResHeight = appSelectedInternalHeight;
if(SL_FAILED(result, slDLSSGSetOptions(viewport, options)))
{
    // Handle error here, check the logs
}
```

Additionally, in development (i.e. non-production) builds of sl.dlss_g.dll, it is possible to enable DLSS-G dynamic res mode globally for debugging purposes via sl.dlss_g.json.  The supported options are:

* `"forceDynamicRes": true,` force-enables DLSS-G dynamic mode, equivalent to passing the flag `eDynamicResolutionEnabled` to `slDLSSGSetOptions` on every frame.
* `"forceDynamicResScaling": 0.5` sets the desired `dynamicResWidth` and `dynamicResHeight` indirectly, as a fraction of the color output buffer size.  In the case shown, the fraction is 0.5, so with a color buffer that is 3840x2160, the internal resolution used by DLSS-G for dynamic resolution MVec and Depth buffers will be 1920x1080.  If this value is not set, it defaults to 0.5.

### 11.0 DLSS-G AND HDR

If your game supports HDR please make sure to use **UINT10/RGB10 pixel format and HDR10/BT.2100 color space**. For more details please see <https://docs.microsoft.com/en-us/windows/win32/direct3darticles/high-dynamic-range#option-2-use-uint10rgb10-pixel-format-and-hdr10bt2100-color-space>

When tagging `eUIColorAndAlpha` please make sure that alpha channel has enough precision (for example do NOT use formats like R10G10B10A2)

> **IMPORTANT:**
> DLSS-G currently does NOT support FP16 pixel format and scRGB color space because it is too expensive in terms of compute and bandwidth cost.

### 12.0 DLSS-G AND DXGI

DLSS-G takes over frame presenting so it is important for the host application to turn on/off DLSS-G as needed to avoid potential problems and deadlocks.
As a general rule, **when host is modifying resolution, full-screen vs windowed mode or performing any other operation that could cause SwapChain::Present call to generate a deadlock DLSS-G must be turned off by the host using the sl::DLSSGConsts::mode field.** When turned off DLSS-G will call SwapChain::Present on the same thread as the host application which is not the case when DLSS-G is turned on. For more details please see <https://docs.microsoft.com/en-us/windows/win32/direct3darticles/dxgi-best-practices#multithreading-and-dxgi>

> **IMPORTANT:**
> Turning DLSS-G on and off using the `sl::DLSSGOptions::mode` should not be confused with enabling/disabling DLSS-G feature using the `slSetFeatureLoaded`, the later would completely unload and unhook the sl.dlss_g plugin hence completely disable the `sl::kFeatureDLSS_G` (cannot be turned on/off or used in any way).

### 13.0 HOW TO OBTAIN THE ACTUAL FRAME TIMES AND NUMBER OF FRAMES PRESENTED

Since DLSS-G when turned on presents additional frames the actual frame time can be obtained using the following sample code:

```cpp

// Using helpers from sl_dlss_g.h

// Not passing flags or special options here, no need since we just want the frame stats
sl::DLSSGState state{};
if(SL_FAILED(result, slDLSSGGetState(viewport, state)))
{
    // Handle error here, check the logs
}
```

> **IMPORTANT:**
> When querying only frame times or status, do not specify the `DLSSGFlags::eRequestVRAMEstimate`; setting that flag and passing a non-null `sl::DLSSGOptions` will cause DLSS-G to compute and return the estimated VRAM required.  This is needless and too expensive to do per frame.

Once we have obtained DLSS-G state we can estimate the actual FPS like this:

```cpp
//! IMPORTANT: Returned value represents number of frames presented since 
//! we last called slDLSSGGetState so make sure to account for that.
//!
//! If calling 'slDLSSGGetState' after each present then the actual FPS
//! can be computed like this:
auto actualFPS = myFPS * state.numFramesActuallyPresented;
```

The `numFramesActuallyPresented` is equal to the number of presented frames per one application frame. For example, if DLSS-G plugin is inserting one generated frame after each application frame, that variable will contain '2'.

> **IMPORTANT**

Please note that DLSS-G will **always present real frame generated by the host but the interpolated frame can be dropped** if presents go out of sync (interpolated frame is too close to the last real one). In addition, if the host is CPU bottlenecked it is **possible for the reported FPS to be more than 2x when DLSS-G is on** because the call to `Swapchain::Present` is no longer a blocking call for the host and can be up to 1ms faster which then translates to faster base frame times. Here is an example:

* Host is CPU bound and producing frames every 10ms
* Up to 1ms is spent blocked by the `Swapchain::Present` call
* SL present hook will take around 0.2ms instead since `Swapchain::Present` is now an async event handled by the SL pacer
* Host is now delivering frames at 10ms - 0.8ms = 9.2ms
* This results in 109fps getting bumped to 218fps when DLSS-G is active so 2.18x scaling instead of the expected 2x

### 14.0 HOW TO CHECK DLSS-G STATUS AT RUNTIME

#### 14.1 HOW TO CHECK FOR MULTIFRAME SUPPORT

Multi-frame support is reported via `sl::DLSSGState::numFramesToGenerateMax`.
Before enabling multi-frame, check for device support by calling
`slDLSSGGetState` and checking `numFramesToGenerateMax`. If the value is 1,
multi-frame is not supported. Otherwise, multi-frame is supported, up to the
number of frames specified.

#### 14.2 HOW TO CHECK FOR RUNTIME ERRORS

Even if DLSS-G feature is supported and loaded it can still end up in an invalid state at run-time due to various reasons. The following code snippet shows how to check the run-time status:

```cpp
sl::DLSSGState state{};
if(SL_FAILED(result, slDLSSGGetState(viewport, state)))
{
    // Handle error here, check the logs
}
// Run-time status
if(state.status != sl::eDLSSGStatusOk)
{
    // Turn off DLSS-G

    sl::DLSSGOptions options{};    
    options.mode = sl::DLSSGMode::eOff;
    slDLSSGSetOptions(viewport, options);
    // Check status and errors in the log and fix your integration if applicable
}
```

For more details please see `enum DLSSGStatus` in sl_dlss_g.h

> **IMPORTANT:**
> When in invalid state and turned on DLSS-G will add pink overlay to the final color image. Warning message will be shown on screen in the NDA development build and error will be logged describing the issue.

> **IMPORTANT:**
> When querying only frame times or status, do not specify the `DLSSGFlags::eRequestVRAMEstimate`; setting that flag and passing a non-null `sl::DLSSGOptions::ext` will cause DLSS-G to compute and return the estimated VRAM required.  This is needless and too expensive to do per frame.

### 15.0 HOW TO GET AN ESTIMATE OF VRAM REQUIRED BY DLSS-G

SL can return a general estimate of the GPU memory required by DLSS-G via `slDLSSGGetState`.  This can be queried before DLSS-G is enabled, and can be queried for resolutions and formats other than those currently active.  To receive an estimate of GPU memory required, the application must:

* Set the `sl::DLSSGOptions::flags` flag, `DLSSGFlags::eRequestVRAMEstimate`
* Provide the values in the `sl::DLSSGOptions` structure include the intended resolutions of the MVecs, Depth buffer, final color buffer (UI buffers are assumed to be the same size as the color buffer), as well as the 3D API-specific format enums for each buffer.  Finally, the expected number of backbuffers in the swapchain must be specified.  See the `sl::DLSSGOptions` struct for details.

If the flag and structure are provided, `slDLSSGGetState` should return a nonzero value in `sl::DLSSGState::estimatedVRAMUsageInBytes`.  Note that this value is a very rough estimate/guideline and should be used for general allocation.  The actual amount used may differ from this value.

> **IMPORTANT:**
> When querying only frame times or status, do not specify the `DLSSGFlags::eRequestVRAMEstimate`; setting that flag and passing a non-null `sl::DLSSGOptions` will cause DLSS-G to compute and return the estimated VRAM required.  This is needless and too expensive to do per frame.

#### 15.1 HOW TO SYNCHRONIZE THE HOST APP DLSS-G INPUTS AND STREAMLINE IF REQUIRED

```cpp
//! SL client must wait on SL DLSS-G plugin-internal fence and associated value, before it can modify or destroy the tagged resources input
//! to DLSS-G enabled for the corresponding previously presented frame on a non-presenting queue.
//! If modified on client's presenting queue, then it's recommended but not required.
//! However, if DLSSGQueueParallelismMode::eBlockNoClientQueues is set, then it's always required for VK.
//! It must call slDLSSGGetState on the present thread to retrieve the fence value for the inputs consumed by FG, on which client would
//! wait in the frame it would modify those inputs.
void* inputsProcessingCompletionFence{};
uint64_t lastPresentInputsProcessingCompletionFenceValue{};
```

### 16.0 HOW TO SYNCHRONIZE THE HOST APP AND STREAMLINE WHEN USING VULKAN

SL DLSS-G implements the following logic when intercepting `vkQueuePresentKHR` and `vkAcquireNextImageKHR`:

* sl.dlssg will wait for the binary semaphore provided in the `VkPresentInfoKHR` before proceeding with adding workload(s) to the GPU
* sl.dlssg will signal binary semaphore provided in `vkAcquireNextImageKHR` call when DLSS-G workloads are submitted to the GPU

Based on this the host application MUST:

* Signal the `present` binary semaphore provided in `VkPresentInfoKHR` when submitting final workload at the end of the frame
* Wait for the signal on the `acquire` binary semaphore provided with `vkAcquireNextImageKHR` call before starting the new frame

Here is some pseudo-code:

```cpp
createBinarySemaphore(acquireSemaphore);
createBinarySemaphore(presentSemaphore);

// SL will signal the 'acquireSemaphore' when ready to continue next frame
vkAcquireNextImageKHR(acquireSemaphore, &index);

// Frame start
waitOnGPU(acquireSemaphore);

// Render frame using render target with given index
renderFrame(index);

// Finish frame
signalOnGPU(presentSemaphore);

// Present the frame (SL will wait for the 'presentSemaphore' on the GPU)
vkQueuePresent(presentSemaphore, index);

```

### 17.0 DLSS-G INTEGRATION CHECKLIST DETAILS

* Provide either correct application ID or engine type (Unity, UE etc.) when calling `slInit`
* In final (production) builds validate the public key for the NVIDIA custom digital certificate on `sl.interposer.dll` if using the binaries provided by NVIDIA. See [security section](ProgrammingGuide.md#211-security) for more details.
* Tag `eDepth`, `eMotionVectors`, `eHUDLessColor` and `eUIColorAndAlpha` buffers
  * When values of depth and mvec could be invalid make sure to set all tags to null pointers (level loading, playing video cut-scenes, paused, in menu etc.)
  * Tagged buffers must by marked as volatile if they are not going to be valid when SwapChain::Present call is made
* Tag backbuffer, only if DLSS-G needs to run on a subregion of the final color buffer. If tagged, ensure to set the tag to null pointer, if it could be invalid.
* Provide correct common constants and frame index using `slSetConstants` method.
  * When game is rendering game frames make sure to set `sl::Constants::renderingGameFrames` correctly
* Make sure that frame index provided with the common constants is matching the presented frame (i.e. frame index provided with Reflex markers `ReflexMarker::ePresentStart` and `ReflexMarker::ePresentEnd`)
* **Do NOT set common constants (camera matrices etc) multiple times per single frame** - this causes ambiguity which can result in IQ issues.
* Use sl.imgui plugin to validate that inputs (camera matrices, depth, mvec, color etc.) are correct
* Turn DLSS-G off (by setting `sl::DLSSGOptions::mode` to `DLSSGMode::eOff`) before any window manipulation (resize, maximize/minimize, full-screen transition etc.) to avoid potential deadlocks or instability
* Reduce the amount of motion blur when DLSS-G is active
* Call `slDLSSGGetState` to obtain `sl::DLSSGState` and check the following:
  * Make sure that `sl::DLSSGStatus` is set to `eDLSSGStatusOk`, if not disable DLSS-G and fix integration as needed (please see the logs for errors)
  * If swap-chain back buffer size is lower than `sl::DLSSGSettings::minWidthOrHeight` DLSS-G must be disabled
  * If VRAM stats and other extra information is not needed pass `nullptr` for constants for lowest overhead.
* Call `slGetFeatureRequirements` to obtain requirements for DLSS-G (see [programming guide](./ProgrammingGuide.md#23-checking-features-requirements) and check the following:
  * If any of the items in the `sl::FeatureRequirements` structure like OS, driver etc. are NOT supported inform user accordingly.
* To avoid an additional overhead when presenting frames while DLSS-G is off **always make sure to re-create the swap-chain when DLSS-G is turned off**. For details please see [section 18](#180-how-to-avoid-unnecessary-overhead-when-dlss-g-is-turned-off)
* `In Vulkan`, to exploit command queue parallelism, setting `DLSSGOptions::queueParallelismMode` to 'DLSSGQueueParallelismMode::eBlockNoClientQueues' mode might offer extra performance gains depending on the workload. 
  * Same DLSSGQueueParallelismMode mode but be set for all the viewports.
  * When using this mode, the client should wait on `DLSSGState::inputsProcessingCompletionFence` and associated value, before client can modify or destroy the tagged resources input to DLSS-G enabled for the corresponding previously presented frame on any of its queues.
  * For synchronization details, please refer [section 16.1](#161-how-to-synchronize-the-host-app-dlss-g-inputs-and-streamline-if-required).
  * Typical scenario in which gains might be more apparent is in GPU-limited applications having workload types employing multiple queues for submissions, especially if the presenting queue is the only one accessing FG inputs. Workloads from other application queues can happen in parallel with the DLSS-G workload; if those workloads underutilize GPU SM resources, the DLSS-G workload may better fill out SM utilization, improving overall performance. On the other hand, highly CPU-limited applications could see relatively smaller gains due to lower parallelism.

#### 17.1 Game setup for the testing DLSS Frame Generation

1. Set up a machine with an Ada board and drivers recommended by NVIDIA team.
1. Turn on Hardware GPU Scheduling: Windows Display Settings (scroll down) -> Graphics Settings -> Hardware-accelerated GPU Scheduling: ON. Restart your PC.
1. Check that Vertical Sync is set to “Use the 3D application setting” in the NVIDIA Control Panel (“Manage 3D Settings”).
1. Get the game build that has Streamline, DLSS-G and Reflex integrated and install on the machine.
1. Once the game has loaded, go into the game settings and turn DLSS-G on.
1. Once DLSS-G is on, you should be able to see it by:
    * observing FPS boost in any external FPS measurement tool; and
    * if the build includes Streamline and DLSS-G development libraries, seeing a debug overlay at the bottom of the screen (can be set in sl.dlss-g.json).

If the steps above fail, set up logging in sl.interposer.json, check for easy-to-fix issues & errors in the log, and contact NVIDIA team.

### 18.0 HOW TO AVOID UNNECESSARY OVERHEAD WHEN DLSS-G IS TURNED OFF

When DLSS-G is loaded it will create an extra graphics command queue used to present frames asynchronously and in addition it will force the host application to render off-screen (host has no access to the swap-chain buffers directly). In scenarios when DLSS-G is switched off by the user
this results in unnecessary overhead coming from the extra copy from the off-screen buffer to the back buffer and synchronization between the game's graphics queue and the DLSS-G's queue. To avoid this, swap-chain must be torn down and re-created every time DLSS-G is switched on or off.

Here is some pseudo code showing how this can be done:

```cpp
void onDLSSGModeChange(sl::DLSSGMode mode)
{
    if(mode == sl::DLSSGMode::eOn || mode == sl::DLSSGMode::eAuto)
    {
        // DLSS-G was off, now we are turning it on or set the mode to auto

        // Make sure no work is pending on GPU
        waitForIdle();
        // Destroy swap-chain back buffers
        releaseBackBuffers();
        // Release swap-chain
        releaseSwapChain();
        // Make sure DLSS-G is loaded
        slSetFeatureLoaded(sl::kFeatureDLSS_G, true);
        // Re-create our swap-chain using the same parameters as before
        // Note that DLSS-G is loaded so SL will return a proxy (assuming host is linking SL and using SL proxy DXGI factory)
        auto swapChainProxy = createSwapChain();
        // Obtain native swap-chain if using manual hooking        
        slGetNativeInterface(swapChainProxy,&swapChainNative);    
        // Obtain new back buffers from the swap-chain proxy (rendering off-screen)
        getBackBuffers(swapChainProxy)
    }
    else if(mode == sl::DLSSGMode::eOff)
    {
        // DLSS-G was on, now we are turning it off

        // Make sure no work is pending on GPU
        waitForIdle();
        // Destroy swap-chain back buffers
        releaseBackBuffers();
        // Release swap-chain
        releaseSwapChain();
        // Make sure DLSS-G is un-loaded
        slSetFeatureLoaded(sl::kFeatureDLSS_G, false);
        // Re-create our swap-chain using the same parameters as before
        // Note that DLSS-G is unloaded so there is no proxy here, SL will return native swap-chain interface
        auto swapChainNative = createSwapChain();
        // Obtain new back buffers from the swap-chain (rendering directly to back buffers)
        getBackBuffers(swapChainNative)
    }    
}
```

For the additional implementation details please check out the Streamline sample, especially the `void DeviceManagerOverride_DX12::BeginFrame()` function.

> NOTE:
> When DLSS-G is turned on the overhead from rendering to an off-screen target is negligible considering the overall frame rate boost provided by the feature.

### 19.0 DLSS-FG INDICATOR TEXT

DLSS-FG can render on-screen indicator text when the feature is enabled. Developers may find this helpful for confirming DLSS-FG is executing.

The indicator supports all build variants, including production.

The indicator is configured via the Windows Registry and contains 3 levels: `{0, 1, 2}` for `{off, minimal, detailed}`.

**Example .reg file setting the level to detailed:**

```
[HKEY_LOCAL_MACHINE\SOFTWARE\NVIDIA Corporation\Global\NGXCore]
"DLSSG_IndicatorText"=dword:00000002
```

### 20.0 AUTO SCENE CHANGE DETECTION

Auto Scene Change Detection (ASCD) intelligently annotates the reset flag during input frame pair sequences.

ASCD is enabled in all DLSS-FG build variants, executes on every frame pair, and supports all graphics platforms.

#### 20.1 INPUT DATA

ASCD uses the camera forward, right, and up vectors passed into Streamline via `sl_consts.h`. These are stitched into a 3x3 camera rotation matrix such that:

```
[ cameraRight[0] cameraUp[0] cameraForward[0] ]
[ cameraRight[1] cameraUp[1] cameraForward[1] ]
[ cameraRight[2] cameraUp[2] cameraForward[2] ]
```

It is important that this matrix is orthonormal, i.e. the transpose of the matrix should equal the inverse. ASCD will only run if the orthonormal property is true. If the orthonormal check fails, ASCD is entirely disabled. Logs for DLSS-FG will show additional detail to debug incorrect input data.

#### 20.2 VIEWING STATUS

In all variants the detector status can be visualized with the detailed DLSS_G Indicator Text.

The mode will be
* Enabled
* Disabled
* Disabled (Invalid Input Data)

In developer builds, ASCD can be toggled with `Shift+F9`. In developer builds, an additional ignore_reset_flag option simulates pure dependence on ASCD `Shift+F10`.

In cases where input camera data is incorrect, ASCD will report failure to the logs every frame. Log messages can be resolved by updating the camera inputs or disabling ASCD temporarily with the keybind.

#### 20.3 DEVELOPER HINTS

In developer DLSS-FG variants ASCD displays on-screen hints for:

1. Scene change detected without the reset flag.
2. Scene change detected with the reset flag.
3. No scene change detected with the reset flag.

The hints present as text blurbs in the center of screen, messages in the DLSS-FG log file, and in scenario 1, a screen goldenrod yellow tint.

### 21.0 DYNAMIC FRAME GENERATION

Dynamic Frame Generation leverages stochastic control to automatically trigger DLSS-G. This adaptive monitoring mechanism activates frame generation only when it boosts performance beyond the native framerate production of the game. Otherwise, DLSS-G remains disabled to ensure optimal framerate performance.

#### 21.1 DLSS-G AUTO MODE

Dynamic Frame Generation is enabled when DLSS-G is in auto mode. To activate Dynamic Frame Generation, set `mode` to `sl::DLSSGMode::eAuto`.

When using non-production (development) builds of `sl.dlss_g.dll`, the status of Dynamic Frame Generation and the current state of DLSS-G is displayed on the DLSS-G status window.

