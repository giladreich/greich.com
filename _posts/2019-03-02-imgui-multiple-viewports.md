---
layout: post
title: Implementing multiple view ports for ImGui using DirectX.
---

# Implementing multiple view ports for ImGui using DirectX

## Introduction

[ImGui](https://github.com/ocornut/imgui) is one of the most user-friendly graphical user interface libraries for C++. I used it in multiple projects and I was impressed with the fact that it does not require any dependencies. There is also a huge community actively contributing to it.

One of ImGui's features allows for dragging child windows outside the main window (similar to the Qt tool bar). This can be useful in several situations: i.e. rendering multiple child windows without having to cover the main scene, allowing them to be dragged in and out conveniently.

![](/assets/img/posts/imgui-multiple-view-ports/multiple-view-ports.gif)

Upon working with DirectX-9 project, I noticed ImGui did not provide implementation for it, so I decided to look at how different graphics APIs were implemented in order to stay consistent with the project coding convention and add my own implementation.

Note that the multiple-view-port feature is not yet merged to the master branch of the repository, switching to the docking branch is required to use it.


## Using multiple view ports

In order to use multiple view ports feature, `ImGuiConfigFlags_ViewportsEnable` flag first needs to be enabled at the initialization phase by using `ImGuiIO`:
```cpp
ImGuiIO& io = ImGui::GetIO();
io.ConfigFlags |= ImGuiConfigFlags_ViewportsEnable;
```

I also set the window style explicitly, so it would look identical in all platforms:
```cpp
ImGuiStyle& style = ImGui::GetStyle();
style.WindowRounding = 0.0f;
style.Colors[ImGuiCol_WindowBg].w = 1.0f;
```

Finally, just before presenting the frame, it is expected to both call `UpdatePlatformWindows` in order to handle the creation/update of all child windows as well as `RenderPlatformWindowsDefault` in order to swap buffers when the child windows is outside the main window:
```cpp
ImGui::UpdatePlatformWindows();
ImGui::RenderPlatformWindowsDefault();

g_pd3dDevice->Present(NULL, NULL, NULL, NULL);
```

This is pretty much all it takes to get started with multiple view ports.

It all sounds simple, right? When it comes to cross-platform code and supporting multiple graphics APIs, this can get a little bit tricky, but with the right abstraction and design, it is possible. I can say for that part, Omar (the author of the project) and the community did an amazing job. I will now explain why.

For every graphic API, there are `imgui_impl_*.*` sources, which take care of the API/Platform specific code in order to abstract all API calls. For instance, when using ImGui with Vulkan API, it is needed to have `imgui_impl_sdl.cpp/h` for wrapping the window creation using SDL and of course `imgui_impl_vulkan.cpp/h` for specific Vulkan API code.

Because this post is about implementing multiple view ports using DirectX-9, all the backend code will go into `imgui_impl_dx9.cpp/h` and `imgui_impl_win32.cpp/h`.


## Implementing multiple view ports

In `imgui_impl_dx9.cpp` these are all the functions that will be implemented required for making multiple view ports work:
```cpp
// Renderer callback declarations 
static void ImGui_ImplDX9_CreateWindow(ImGuiViewport* viewport);
static void ImGui_ImplDX9_DestroyWindow(ImGuiViewport* viewport);
static void ImGui_ImplDX9_SetWindowSize(ImGuiViewport* viewport, ImVec2 size);
static void ImGui_ImplDX9_RenderWindow(ImGuiViewport* viewport, void*);
static void ImGui_ImplDX9_SwapBuffers(ImGuiViewport* viewport, void*);

// Additional related declarations
static void ImGui_ImplDX9_InitPlatformInterface();
static void ImGui_ImplDX9_ShutdownPlatformInterface();
static void ImGui_ImplDX9_CreateDeviceObjectsForPlatformWindows();
static void ImGui_ImplDX9_InvalidateDeviceObjectsForPlatformWindows();
```

Every renderer function receives `ImGuiViewport` pointer and must be matched with the declaration in `ImGuiPlatformIO` interface. This is exactly a small fraction of the abstraction I was previously talking about. Here is the definition of `ImGuiViewport` structure in `imgui.h`:
```cpp
struct ImGuiViewport
{
    ImGuiID             ID;                     
    ImGuiViewportFlags  Flags;                  
    ImVec2              Pos;                    
    ImVec2              Size;                   
    float               DpiScale;               
    ImDrawData*         DrawData;               
    ImGuiID             ParentViewportId;       

    void*               RendererUserData;       
    void*               PlatformUserData;       
    void*               PlatformHandle;         
    void*               PlatformHandleRaw;      
    bool                PlatformRequestClose;   
    bool                PlatformRequestMove;    
    bool                PlatformRequestResize;  

    ImGuiViewport()     { ID = 0; Flags = 0; DpiScale = 0.0f; DrawData = NULL; ParentViewportId = 0; RendererUserData = PlatformUserData = PlatformHandle = PlatformHandleRaw = NULL; PlatformRequestClose = PlatformRequestMove = PlatformRequestResize = false; }
    ~ImGuiViewport()    { IM_ASSERT(PlatformUserData == NULL && RendererUserData == NULL); }
};
```

The way this works, in `imgui.cpp` (where it is not specific graphic API code), it iterates through all viewports to pass the calls forward. Here is a small snippet of `UpdatePlatformWindows` to get the idea:
```cpp
void ImGui::UpdatePlatformWindows()
{
    ImGuiContext& context = *GImGui;
    ImGuiPlatformIO* platform = &context.PlatformIO;
    for (int i = 1; i < context.Viewports.Size; i++)
    {
        ImGuiViewportP* viewport = context.Viewports[i];
        if (platform->Renderer_CreateWindow)
            platform->Renderer_CreateWindow(viewport);
    }
}
``` 

For every iteration, it checks to see whether the function pointer is valid. If so, a pointer to the current viewport is passed so the consumer can write specific graphic API code. The same behavior happens with all other `ImGuiPlatformIO` functions.

In the `ImGuiViewport` structure, there are multiple void pointers that are set differently depending on the graphic API. As every API provides a different structure definition and implementation, there is no guarantee that they will always be the same size, thus they are set to void*. In this example, I will focus on the `void* RendererUserData` where I set specific graphic API (DirectX-9) structure into renderer data.

In order for it to be able to render to multiple windows, it needs to allocate memory for additional swap-chain. It is therefore necessary to provide specified graphic API structure definition:
```cpp
struct ImGuiViewportDataDx9
{
    IDirect3DSwapChain9*    SwapChain;
    D3DPRESENT_PARAMETERS   d3dpp;

    ImGuiViewportDataDx9()  { SwapChain = NULL; ZeroMemory(&d3dpp, sizeof(D3DPRESENT_PARAMETERS)); }
    ~ImGuiViewportDataDx9() { IM_ASSERT(SwapChain == NULL); }
};
```

First step is to reference all specific DirectX-9 API implementation using `ImGuiPlatformIO` during the initialization phase:
```cpp
static void ImGui_ImplDX9_InitPlatformInterface()
{
    ImGuiPlatformIO& platform_io = ImGui::GetPlatformIO();
    platform_io.Renderer_CreateWindow  = ImGui_ImplDX9_CreateWindow;
    platform_io.Renderer_DestroyWindow = ImGui_ImplDX9_DestroyWindow;
    platform_io.Renderer_SetWindowSize = ImGui_ImplDX9_SetWindowSize;
    platform_io.Renderer_RenderWindow  = ImGui_ImplDX9_RenderWindow;
    platform_io.Renderer_SwapBuffers   = ImGui_ImplDX9_SwapBuffers;
}
```

During runtime, whenever a child window is dragged outside the main window, `ImGui_ImplDX9_CreateWindow` gets called to allocate memory for the creation of additional swap-chain and present parameters description ([D3DPRESENT_PARAMETERS](https://docs.microsoft.com/en-us/windows/win32/direct3d9/d3dpresent-parameters)):
```cpp
static void ImGui_ImplDX9_CreateWindow(ImGuiViewport* viewport)
{
    // After allocating memory, we set the data to the renderer void * so it is accessible.
    ImGuiViewportDataDx9* data = IM_NEW(ImGuiViewportDataDx9)();
    viewport->RendererUserData = data;

    // Making sure that we have a handle to the window, so we can use it to create the swap-chain.
    HWND hwnd = viewport->PlatformHandleRaw ? (HWND)viewport->PlatformHandleRaw : (HWND)viewport->PlatformHandle;
    IM_ASSERT(hwnd != NULL);

    // Describe present parameters.
    ZeroMemory(&data->d3dpp, sizeof(D3DPRESENT_PARAMETERS));
    data->d3dpp.Windowed = TRUE;
    data->d3dpp.SwapEffect = D3DSWAPEFFECT_DISCARD;
    data->d3dpp.BackBufferWidth = (UINT)viewport->Size.x;
    data->d3dpp.BackBufferHeight = (UINT)viewport->Size.y;
    data->d3dpp.BackBufferFormat = D3DFMT_UNKNOWN;
    data->d3dpp.hDeviceWindow = hwnd;
    data->d3dpp.EnableAutoDepthStencil = FALSE;
    data->d3dpp.AutoDepthStencilFormat = D3DFMT_D16;
    data->d3dpp.PresentationInterval = D3DPRESENT_INTERVAL_IMMEDIATE;   // Present without vsync
    
    // Create additional swap-chain with the present parameters so we can render to multiple windows.
    HRESULT hr = g_pd3dDevice->CreateAdditionalSwapChain(&data->d3dpp, &data->SwapChain); IM_UNUSED(hr);
    IM_ASSERT(hr == D3D_OK);
    IM_ASSERT(data->SwapChain != NULL);
}
```

Whenever there is an additional window created/dragged outside the main-window, it is time to draw into it from the additional created swap-chain. It is therefore necessary to swap the back buffers for every frame by presenting the frame for each swap-chain:
```cpp
static void ImGui_ImplDX9_SwapBuffers(ImGuiViewport* viewport, void*)
{
    ImGuiViewportDataDx9* data = (ImGuiViewportDataDx9*)viewport->RendererUserData;
    HRESULT hr = data->SwapChain->Present(NULL, NULL, data->d3dpp.hDeviceWindow, NULL, NULL);
    IM_ASSERT(hr == D3D_OK);
}
```

It can now be seen that for every viewport, it is possible to safely cast it to a specific graphic API structure. In this example, it is the previously new structure added for DirectX-9 `ImGuiViewportDataDx9` containing the swap-chain that can be used in order to present the frame.

For the rendering part, after clearing the background color and doing preparation before drawing the UIs, it is now time to pass the viewport draw data to `ImGui_ImplDX9_RenderDrawData`:
```cpp
static void ImGui_ImplDX9_RenderWindow(ImGuiViewport* viewport, void*)
{
    ImGuiViewportDataDx9* data = (ImGuiViewportDataDx9*)viewport->RendererUserData;
    ImVec4 clear_color = ImVec4(0.0f, 0.0f, 0.0f, 1.0f);

    LPDIRECT3DSURFACE9 render_target = NULL;
    LPDIRECT3DSURFACE9 last_render_target = NULL;
    LPDIRECT3DSURFACE9 last_depth_stencil = NULL;
    data->SwapChain->GetBackBuffer(0, D3DBACKBUFFER_TYPE_MONO, &render_target);
    g_pd3dDevice->GetRenderTarget(0, &last_render_target);
    g_pd3dDevice->GetDepthStencilSurface(&last_depth_stencil);
    g_pd3dDevice->SetRenderTarget(0, render_target);
    g_pd3dDevice->SetDepthStencilSurface(NULL);

    if (!(viewport->Flags & ImGuiViewportFlags_NoRendererClear))
    {
        D3DCOLOR clear_col_dx = D3DCOLOR_RGBA((int)(clear_color.x*255.0f), (int)(clear_color.y*255.0f), (int)(clear_color.z*255.0f), (int)(clear_color.w*255.0f));
        g_pd3dDevice->Clear(0, NULL, D3DCLEAR_TARGET, clear_col_dx, 1.0f, 0);
    }

    ImGui_ImplDX9_RenderDrawData(viewport->DrawData);

    // Restore render target
    g_pd3dDevice->SetRenderTarget(0, last_render_target);
    g_pd3dDevice->SetDepthStencilSurface(last_depth_stencil);
    render_target->Release();
    last_render_target->Release();
    if (last_depth_stencil) last_depth_stencil->Release();
}
```

Note that it is also necessary to backup and restore every viewport render-target and depth-stencil to avoid conflicts with the global depth-stencil configurations. This would at times cause the UIs, that are drawn outside the main-window, to be black.

In handling resize events, the `ImGui_ImplDX9_SetWindowSize` function takes care of recreating the swapchain with the specified width and height portions:
```cpp
static void ImGui_ImplDX9_SetWindowSize(ImGuiViewport* viewport, ImVec2 size)
{
    ImGuiViewportDataDx9* data = (ImGuiViewportDataDx9*)viewport->RendererUserData;
    if (data->SwapChain)
    {
        data->SwapChain->Release();
        data->SwapChain = NULL;
        data->d3dpp.BackBufferWidth = (UINT)size.x;
        data->d3dpp.BackBufferHeight = (UINT)size.y;
        HRESULT hr = g_pd3dDevice->CreateAdditionalSwapChain(&data->d3dpp, &data->SwapChain); IM_UNUSED(hr);
        IM_ASSERT(hr == D3D_OK);
    }
}
```

Note that in DirectX >= 10 it is as simple as calling [ResizeBuffers](https://docs.microsoft.com/en-us/windows/win32/api/dxgi/nf-dxgi-idxgiswapchain-resizebuffers). As I could not find another way of resizing the back-buffers with DirectX-9, I recreated the swap-chain with the new size for every size-changed event, which works well.

In the event of a child-window returning to the main-window, it is necessary to properly destroy and release any referenced resources to avoid memory leaks:
```cpp
static void ImGui_ImplDX9_DestroyWindow(ImGuiViewport* viewport)
{
    if (ImGuiViewportDataDx9* data = (ImGuiViewportDataDx9*)viewport->RendererUserData)
    {
        if (data->SwapChain)
            data->SwapChain->Release();
        data->SwapChain = NULL;
        ZeroMemory(&data->d3dpp, sizeof(D3DPRESENT_PARAMETERS));
        IM_DELETE(data);
    }
    viewport->RendererUserData = NULL;
}
```

Last but not least, as the application closes, calling `ImGui_ImplDX9_ShutdownPlatformInterface` will destroy all viewports as well as releasing all referenced resources.
