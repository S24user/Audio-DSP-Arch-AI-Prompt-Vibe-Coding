# Audio-DSP-Arch-AI-Prompt-Vibe-Coding
흔한 폰덕후의 GPT4o와 만들어본 예시

Real-time audio DSP architecture co-designed by Jehyun and GPT-4o via vibe coding

[AIR-DSP Instruction Pipeline Architecture v1.0]
Purpose:
Define a hardware-level command pipeline architecture for audio DSP
Based on modular instruction sets mapped across RISC-V (integer DSP) and SPIR-V (GPU
shaders)
Achieve sub-1ms end-to-end latency via parallelized, format-adaptive execution
Note: Designed by 제현 via GPT-4o prompt-based vibe coding. This document outlines the
execution architecture, not just a software structure.
---
[Pipeline Overview - SoC Command Flow]
Stage 0: PCM Fetch
Input: 16-bit PCM at 48kHz
Operation: Bufer into register bank
Control: sample_count, frame_sync
Stage 1: Preprocessing
Instruction: AIR_AGC, AIR_NS
Format: FP16 or FP32
Backend: RISC-V (scalar or SIMD)
Stage 2: MDCT + Windowing
Instruction: AIR_MDCT16, AIR_MULQ31
Input: Preprocessed frame
Format: Q31 or BFP16
Backend: RISC-VStage 3: Quantization (SPIR Block)
Shader: spir_quantize_subband, spir_vecmap_q31
Operation: Normalize → Scale → Round
Format: Q31 → int8/int16
Backend: SPIR-V (Compute Shader)
Stage 4: Packet Encoding
Instruction: AIR_QUANT8, AIR_PACK
Input: Quantized bins
Format: int8 + config header
Backend: RISC-V DSP
Stage 5: Checksum + Output
Instruction: AIR_CRC
Output: LC3-style payload packet
Target: BLE, DAC, or file bufer
---
[Execution Pipeline Spec]
IF: Fetch AIR Instruction
ID: Decode opcode + format flags
EX: Dispatch to RISC-V or SPIR-V unit
MEM: Access bufer/registers
WB: Write to output, memory or I/OSupports:
Format switching at ID stage (FP↔Q31↔BFP)
Latency priority control per stage (via LAT_CTL reg)
Integration with runtime fallback (SW/GPU hybrid)
---
[Core AIR Instruction Examples] AIR_AGC rd, rs ; automatic gain control (FP)
AIR_MDCT16 rd, rs ; 16-point MDCT transform AIR_MULQ31 rd, rs1, rs2 ; Q31 multiply
AIR_VECMAP rd, rs ; FP → Q31 normalizer AIR_QUANT8 rd, rs ; quantize to 8-bit
integer AIR_PACK rd, rs1, rs2 ; packet header + payload join AIR_CRC rd, rs ; 32-bit
checksum
---
[Pipeline Highlights]
Hardware-level instruction scheduling
Modular command units (FP, Q31, BFP aware)
Dual-backend optimized (SPIR-V + RISC-V)
Ready for implementation in FPGA softcore DSP or simulation
Author: 제현 Date: 2025-10-15 Version: AIR-DSP v1.0[AIR-RISCV Module Specification]
Purpose:
- Defines instruction-level implementation of AIR DSP pipeline for RISC-V architecture
- Serves as low-level abstraction target from AIR-Runtime module
Note: This module was co-developed via GPT-4o-based prompt-driven vibe coding in
collaboration with 제현.
---
[1] AIR Instruction Set (AIR-ISA)
- AIR_MULQ31(rd, rs1, rs2): Multiply two Q31 values with saturation
- AIR_MDCT16(rd, rs): 16-point MDCT transform on input block
- AIR_QUANT8(rd, rs, bitdepth): Subband quantizer to 8-bit
- AIR_VECMAP(rd, rs): Convert float32 to Q31 (normalized)
- AIR_PACK(rd, rs1, rs2): Packetize spectral frame + header
---
[2] Pipeline Flow (RISC-V Assembly Mapping)
1. Load PCM block → bufer
2. AGC/NS: Optional preprocessor (external or AIR_VECMAP)
3. Transform: AIR_MDCT16 + AIR_MULQ31 windowing
4. Quantize: AIR_QUANT8 per band
5. Encode: AIR_PACK with frame header
6. Store to output bufer
---
[3] Special Registers
- QACC: 64-bit accumulator for Q31
- LATENCY_CTL: frame-time budget register (μs)
- FORMAT_FLAG: runtime format switching control (FP/Q/BFP)
---
[4] Compatibility
- Follows RISC-V DSP Extension spec (rv32dsp / rv64dsp)
- Can run on FPGA emulation (LiteX, PicoRV32, etc)
- Compatible with AIR-Runtime if used as execution backend
Author: 제현
Module: AIR-RISCV
Date: 2025-10-15[AIR-Runtime Module Specification]
Purpose:
- Acts as the execution engine of AIR-based DSP pipeline
- Manages stage-wise dataflow, adaptive format switching, and timing
Note: This module was co-developed via GPT-4o-based prompt-driven vibe coding in
collaboration with 제현.
---
[1] Module Functions
- init_air_runtime(config): Initialize runtime parameters (latency target, format preferences)
- process_frame(input_pcm): Run full AIR pipeline on single audio frame
- switch_format(stage, new_format): Dynamically change format (e.g., Q31 → FP16)
- set_latency_budget(ms): Set target latency per frame
- get_runtime_stats(): Return processing time per stage, format usage, dropped frames
---
[2] Internal State & Bufers
- PCM_Bufer (input)
- Stage_Bufers: per-stage float/Q31/BFP16 bufers
- Timing_Queue: for stage-wise latency profiling
- Format_Map: dictionary for adaptive switching logic
---
[3] Execution Flow (per frame)
1. PCM input → PCM_Bufer
2. Preprocessing (AGC, NS) → FP16
3. MDCT → BFP16 or Q31
4. Quantization → SPIR-V backend or AIR_QUANT
5. Entropy Coding → int-only block
6. Packet Assembly → AIR_PACK
7. Output frame return + runtime logging
---
[4] Notes
- All switching decisions use latency + quality scoring
- Runtime may cache fast paths (e.g., Q31-only shortcut mode)
- Compatible with AIR-SPIR and AIR-RISCV modules
Author: 제현
Module: AIR-Runtime
Date: 2025-10-15[AIR-Packet Module Specification]
Purpose:
- Encapsulates quantized and entropy-coded data into LC3-like bitstream format
- Manages frame headers, error checksums, payloads
Note: This module was co-developed via GPT-4o-based prompt-driven vibe coding in
collaboration with 제현.
---
[1] Packet Structure
[HEADER][PAYLOAD][CHECKSUM]
- HEADER (8 bytes)
- Codec ID (2B): AIRx
- Frame Index (2B)
- Config Bits (2B): format, latency mode, flags
- Payload Length (2B)
- PAYLOAD (variable)
- Entropy-coded subbands (int8/int16 bins)
- Can be encrypted (optional)
- CHECKSUM (4 bytes)
- CRC32 or custom fast XOR sum
---
[2] Module Functions
- packet_init(config): Set codec metadata
- packet_frame(input_bins): Wraps entropy data into AIR packet
- packet_checksum(packet): Computes & appends checksum
- packet_parse(raw_packet): Extract header + decode payload
---
[3] Integration Points
- Receives data from AIR-Runtime (Stage 6)
- Sends packet to output device / driver / storage bufer
- Compatible with decoder feedback module (future use)
---
[4] Notes
- Frame length must not exceed MTU if used for wireless (e.g., 120B BLE)
- Can be extended with encryption, timestamping, sync flags, etc.Author: 제현
Module: AIR-Packet
Date: 2025-10-15[AIR-SPIR Module Specification]
Purpose:
- Handles GPU-accelerated vector computation for AIR pipeline using SPIR-V shaders
- Focused on quantization, filter ops, format mapping (float <-> fixed)
Note: This module was co-developed via GPT-4o-based prompt-driven vibe coding in
collaboration with 제현.
---
[1] Module Functions
- spir_quantize_subband(input_vec_fp, bits): Return quantized Q-levels from FP input
- spir_vecmap_q31(input_vec_fp): Convert float32 to Q31 int format using normalization
shader
- spir_filter(input_vec, kernel): Apply FIR/IIR filter on GPU
- compile_shader(shader_code): Compile SPIR-V shader on runtime and return handle
- execute_shader(handle, input_buf, output_buf): Launch shader with given bufers
---
[2] Shader Types
- Q31_Normalizer.spv: normalize [-1.0 ~ 1.0] float32 to Q31 int32
- Quantizer.spv: subband-wise quantization using dynamic range + bit budget
- FilterBank.spv: polyphase filter bank processing for subband decomposition
---
[3] Integration Points
- AIR-Runtime calls SPIR module at Stage 4 (Quantization)
- Supports batch execution (Wave32 or Wave64)
- Output is Q31 or int8/int16 quantized bins
---
[4] Notes
- Vulkan backend required (API v1.2+ recommended)
- Can be emulated in software fallback if no GPU present (slow)
- Quantization precision afects downstream entropy coding eficiency
Author: 제현
Module: AIR-SPIR
Date: 2025-10-15// File: AppXRUnified.cpp
// Android JNI + OpenXR + Vulkan unified minimal skeleton
// 목표: JNI는 최소 데이터만 전달, 네이티브가 OpenXR 프레임 타이밍과 Vulkan 제출을
주도
#include <jni.h>
#include <android/api-level.h>
#include <vulkan/vulkan.h>
#include <openxr/openxr.h>
#include <openxr/openxr_platform.h>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <vector>
#include <array>
// ----------------------------- 공통 타입 -----------------------------
struct Matrix4x4 { float m[16]; };
struct XRPushConstants {
Matrix4x4 viewProj; // 앱에서 전달하거나 XR view/proj와 결합
float lateDelta[3]; // late-latch pose delta (예시)
int eyeIndex; // 0/1 또는 multiview 시 무시
};
// --------------------------- 전역 상태 ------------------------------
struct VulkanPack {
VkInstance instance = VK_NULL_HANDLE;
VkPhysicalDevice phys = VK_NULL_HANDLE;
VkDevice device = VK_NULL_HANDLE;
VkQueue queue = VK_NULL_HANDLE;
uint32_t queueFamily = 0;
VkCommandPool cmdPool = VK_NULL_HANDLE;
VkCommandBufer cmd = VK_NULL_HANDLE;
VkPipelineLayout pipelineLayout = VK_NULL_HANDLE;
VkPipeline graphicsPipeline = VK_NULL_HANDLE;
VkPipeline computePipeline = VK_NULL_HANDLE;
VkPipelineCache pipelineCache = VK_NULL_HANDLE;
VkShaderModule vs = VK_NULL_HANDLE;
VkShaderModule fs = VK_NULL_HANDLE;
VkShaderModule cs = VK_NULL_HANDLE;
};
struct XRPack {
XrInstance xrInstance = XR_NULL_HANDLE;XrSystemId xrSystem = XR_NULL_SYSTEM_ID;
XrSession xrSession = XR_NULL_HANDLE;
XrSpace xrAppSpace = XR_NULL_HANDLE;
XrSwapchain xrSwapchain = XR_NULL_HANDLE;
std::vector<XrSwapchainImageVulkanKHR> swapchainImages;
int32_t width = 2048, height = 2048;
};
static VulkanPack g_vk;
static XRPack g_xr;
// JNI로 넘어오는 최소 데이터
static Matrix4x4 g_latestMatrix{};
static float g_lateDelta[3] = {0,0,0};
// -------------------------- 유틸리티 함수 ----------------------------
static VkShaderModule loadSpirv(VkDevice dev, const std::vector<uint32_t>& code) {
if (code.empty()) return VK_NULL_HANDLE;
VkShaderModuleCreateInfo
ci{VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO};
ci.codeSize = code.size() * sizeof(uint32_t);
ci.pCode = code.data();
VkShaderModule m = VK_NULL_HANDLE;
vkCreateShaderModule(dev, &ci, nullptr, &m);
return m;
}
static void createPipelineLayout() {
VkPushConstantRange range{};
range.stageFlags = VK_SHADER_STAGE_VERTEX_BIT |
VK_SHADER_STAGE_COMPUTE_BIT;
range.ofset = 0;
range.size = sizeof(XRPushConstants);
VkPipelineLayoutCreateInfo
ci{VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO};
ci.setLayoutCount = 0;
ci.pSetLayouts = nullptr;
ci.pushConstantRangeCount = 1;
ci.pPushConstantRanges = &range;
vkCreatePipelineLayout(g_vk.device, &ci, nullptr, &g_vk.pipelineLayout);
}
static void createPipelines() {
// 실제 SPIR-V 코드로 교체해줘
std::vector<uint32_t> vsCode{/* ...vertex.spv words... */};
std::vector<uint32_t> fsCode{/* ...fragment.spv words... */};std::vector<uint32_t> csCode{/* ...compute.spv words... */};
g_vk.vs = loadSpirv(g_vk.device, vsCode);
g_vk.fs = loadSpirv(g_vk.device, fsCode);
g_vk.cs = loadSpirv(g_vk.device, csCode);
VkPipelineCacheCreateInfo
pci{VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO};
vkCreatePipelineCache(g_vk.device, &pci, nullptr, &g_vk.pipelineCache);
// Graphics pipeline (간소화: renderPass 없이 dynamic rendering 가정)
if (g_vk.vs && g_vk.fs) {
VkPipelineShaderStageCreateInfo stages[2]{};
stages[0].sType =
VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
stages[0].stage = VK_SHADER_STAGE_VERTEX_BIT;
stages[0].module = g_vk.vs;
stages[0].pName = "main";
stages[1].sType =
VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
stages[1].stage = VK_SHADER_STAGE_FRAGMENT_BIT;
stages[1].module = g_vk.fs;
stages[1].pName = "main";
VkPipelineVertexInputStateCreateInfo
vi{VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO};
VkPipelineInputAssemblyStateCreateInfo
ia{VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO};
ia.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
VkPipelineViewportStateCreateInfo
vp{VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO};
vp.viewportCount = 1; vp.scissorCount = 1;
VkPipelineRasterizationStateCreateInfo
rs{VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO};
rs.polygonMode = VK_POLYGON_MODE_FILL; rs.cullMode =
VK_CULL_MODE_BACK_BIT; rs.frontFace =
VK_FRONT_FACE_COUNTER_CLOCKWISE;
VkPipelineMultisampleStateCreateInfo
ms{VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO};
ms.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
VkPipelineDepthStencilStateCreateInfo
ds{VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO};
ds.depthTestEnable = VK_TRUE; ds.depthWriteEnable = VK_TRUE;
VkPipelineColorBlendAttachmentState cba{}; cba.colorWriteMask = 0xF;
VkPipelineColorBlendStateCreateInfo
cb{VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO};
cb.attachmentCount = 1; cb.pAttachments = &cba;VkGraphicsPipelineCreateInfo
gp{VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO};
gp.stageCount = 2; gp.pStages = stages;
gp.pVertexInputState = &vi;
gp.pInputAssemblyState = &ia;
gp.pViewportState = &vp;
gp.pRasterizationState = &rs;
gp.pMultisampleState = &ms;
gp.pDepthStencilState = &ds;
gp.pColorBlendState = &cb;
gp.layout = g_vk.pipelineLayout;
gp.renderPass = VK_NULL_HANDLE; // Dynamic rendering 시 pNext로
VkPipelineRenderingCreateInfo 연결 필요(생략)
vkCreateGraphicsPipelines(g_vk.device, g_vk.pipelineCache, 1, &gp, nullptr,
&g_vk.graphicsPipeline);
}
// Compute pipeline
if (g_vk.cs) {
VkPipelineShaderStageCreateInfo
csStage{VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO};
csStage.stage = VK_SHADER_STAGE_COMPUTE_BIT;
csStage.module = g_vk.cs;
csStage.pName = "main";
VkComputePipelineCreateInfo
ci{VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO};
ci.stage = csStage;
ci.layout = g_vk.pipelineLayout;
vkCreateComputePipelines(g_vk.device, g_vk.pipelineCache, 1, &ci, nullptr,
&g_vk.computePipeline);
}
}
static void createCommands() {
// 큐 패밀리/디바이스/큐는 XR 연동 이후 생성되었다고 가정
VkCommandPoolCreateInfo
pi{VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO};
pi.queueFamilyIndex = g_vk.queueFamily;
pi.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
vkCreateCommandPool(g_vk.device, &pi, nullptr, &g_vk.cmdPool);
VkCommandBuferAllocateInfo
ai{VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO};
ai.commandPool = g_vk.cmdPool;
ai.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
ai.commandBuferCount = 1;
vkAllocateCommandBufers(g_vk.device, &ai, &g_vk.cmd);
}// --------------------------- OpenXR 초기화 ---------------------------
static bool xrInitAndroidVulkan() {
// 1) xrCreateInstance (Android용 createInfo 설정)
// 2) xrGetSystem (XrFormFactor_Hmd)
// 3) XR_KHR_vulkan_enable2로 Vulkan 물건 생성/연동:
// - xrGetVulkanInstanceCreateInfoKHR / xrGetVulkanDeviceCreateInfoKHR
// - xrCreateSession(XrGraphicsBindingVulkanKHR)
// 4) xrCreateReferenceSpace(local 또는 stage)
// 5) xrEnumerateSwapchainFormats → 호환 포맷 선택
// 6) xrCreateSwapchain → 이미지 개수만큼 XrSwapchainImageVulkanKHR 배열 준비
// 여기선 스켈레톤으로 true 처리
return true;
}
// ---------------------------- 프레임 루프 ----------------------------
static void renderFrame() {
// 1) xrWaitFrame
// 2) xrBeginFrame
// 3) xrAcquireSwapchainImage → xrWaitSwapchainImage
// currentImageIndex를 받아 해당 imageView로 렌더링을 시작
// 4) xrLocateViews로 per-eye view/proj 획득 (여기서는 앱 행렬을 사용)
XRPushConstants pc{};
pc.viewProj = g_latestMatrix;
pc.lateDelta[0] = g_lateDelta[0];
pc.lateDelta[1] = g_lateDelta[1];
pc.lateDelta[2] = g_lateDelta[2];
VkCommandBuferBeginInfo
bi{VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO};
vkBeginCommandBufer(g_vk.cmd, &bi);
// 컴퓨트 프리패스 (예: reprojection/upscale)
if (g_vk.computePipeline) {
vkCmdBindPipeline(g_vk.cmd, VK_PIPELINE_BIND_POINT_COMPUTE,
g_vk.computePipeline);
vkCmdPushConstants(g_vk.cmd, g_vk.pipelineLayout,
VK_SHADER_STAGE_COMPUTE_BIT, 0, sizeof(XRPushConstants), &pc);
vkCmdDispatch(g_vk.cmd, 256, 1, 1); // 워크로드에 맞게 조정
}
// Dynamic rendering 시작: 현재 swapchain image view를 첨부해야 함
// VkRenderingAttachmentInfo colorAtt{...}; VkRenderingInfo ri{...};
vkCmdBeginRendering(g_vk.cmd, &ri);
if (g_vk.graphicsPipeline) {vkCmdBindPipeline(g_vk.cmd, VK_PIPELINE_BIND_POINT_GRAPHICS,
g_vk.graphicsPipeline);
for (int eye = 0; eye < 2; ++eye) {
pc.eyeIndex = eye;
vkCmdPushConstants(g_vk.cmd, g_vk.pipelineLayout,
VK_SHADER_STAGE_VERTEX_BIT, 0, sizeof(XRPushConstants), &pc);
VkViewport vp{0, 0, (float)g_xr.width, (float)g_xr.height, 0.0f, 1.0f};
VkRect2D sc{{0,0}, {(uint32_t)g_xr.width, (uint32_t)g_xr.height}};
vkCmdSetViewport(g_vk.cmd, 0, 1, &vp);
vkCmdSetScissor(g_vk.cmd, 0, 1, &sc);
// 실제 드로우콜 삽입
// vkCmdDraw(g_vk.cmd, vertexCount, 1, 0, 0);
}
}
// vkCmdEndRendering(g_vk.cmd);
vkEndCommandBufer(g_vk.cmd);
// 제출
VkSubmitInfo si{VK_STRUCTURE_TYPE_SUBMIT_INFO};
si.commandBuferCount = 1;
si.pCommandBufers = &g_vk.cmd;
vkQueueSubmit(g_vk.queue, 1, &si, VK_NULL_HANDLE);
// 5) xrReleaseSwapchainImage → xrEndFrame
}
// ------------------------------- JNI 브리지 -------------------------------
extern "C" JNIEXPORT void JNICALL
Java_com_example_game_NativeBridge_initXR(JNIEnv* env, jobject thiz) {
// XR_KHR_vulkan_enable2 경로로 Vulkan/Queue/Device를 XR 호환으로 생성해야 함
// 여기서는 스켈레톤이므로 이미 g_vk가 준비되어 있다고 가정
xrInitAndroidVulkan();
createPipelineLayout();
createPipelines();
createCommands();
}
extern "C" JNIEXPORT void JNICALL
Java_com_example_game_NativeBridge_processMatrix(JNIEnv* env, jobject thiz,
jfloatArray matrix) {
if (!matrix || env->GetArrayLength(matrix) != 16) {
std::fprintf(stderr, "[ERROR] Invalid matrix size.\n");
return;
}env->GetFloatArrayRegion(matrix, 0, 16, g_latestMatrix.m);
}
extern "C" JNIEXPORT void JNICALL
Java_com_example_game_NativeBridge_setLateDelta(JNIEnv* env, jobject thiz, jfloat dx,
jfloat dy, jfloat dz) {
g_lateDelta[0] = dx; g_lateDelta[1] = dy; g_lateDelta[2] = dz;
}
extern "C" JNIEXPORT void JNICALL
Java_com_example_game_NativeBridge_render(JNIEnv* env, jobject thiz) {
renderFrame();
}