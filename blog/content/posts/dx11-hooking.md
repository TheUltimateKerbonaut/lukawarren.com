---
title: "Hooking DirectX 11 to create an overlay for modding"
date: 2020-10-31T10:01:56
draft: false
description: "Using ImGui and Detours, drawing a debugging overlay for use in any DirectX 11 game."
slug: "" 
tags: []
categories: []
externalLink: ""
series: []
---

![ImGui overlay over Saints Row the Third](/img/dx11-hooking/ImGui.png)

After messing around with DirectX 11 the other week, I became curious as to how overlays, such as Fraps or those in game mods, worked. So I decided to create my own overlay for Saints Row the Third, a game which used the API, and draw a debug panel with ImGui.

It turns out that to modify the drawing process in a game, I'd have to hook into D3D11, where the drawing loop typically goes along the lines of:
```
ID3D11Device::ClearRenderTargetView(...) // Clear screen
ID3D11DeviceContext::Draw(...) // Draw stuff
IDXGISwapChain::Present(...) // Present the rendered image onto the screen
```

The main goal was to retrieve a pointer to the game's `IDXGISwapChain`, and then use it inside the game's rendering process to draw stuff.

The best function for this would be `IDXGISwapChain::Present`, where the underlying function looked something like:

```
HRESULT(__stdcall* IDXGISwapChainPresent)(IDXGISwapChain* pSwapChain, UINT SyncInterval, UINT Flags)
{
	...
}
```

If I could execute my own code inside that function when it was called, I could render whatever I pleased, before resuming control to DirectX. The process went a little something like:

* Use a dummy D3D11 device to get the address of IDXGISwapChainPresent
* Trampoline hook the function
* Inside the function, get a copy of the device and swap chain
* Redirect input as well to handle mouse and keyboard events
* Draw my overlay every frame before handing back control


# Getting a foothold
The first step was to create a new C++ application that compiled to a DLL and could be injected into the process.

```
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ulReasonForCall, LPVOID lpReserved)
{
	// Sanity check
	if (!GetModuleHandle(L"SaintsRowTheThird_DX11.exe")) return FALSE;

    switch (ulReasonForCall)
    {
        case DLL_PROCESS_ATTACH:
			// Spawn new thread
            CloseHandle(CreateThread(nullptr, 0, (LPTHREAD_START_ROUTINE)SpawnConsoleThread, hModule, 0, nullptr));
        break;
        case DLL_THREAD_ATTACH:

        break;
        case DLL_THREAD_DETACH:

        break;
        case DLL_PROCESS_DETACH:

        break;
    }

    return TRUE;
}
```

Inside my new thread, I spawned a command prompt and redirected stdout and stderr for debugging purposes, then checked D3D11 was actually present.

```
DWORD WINAPI SpawnConsoleThread(HMODULE hModule)
{
	// Allocate a new console for the calling process
	AllocConsole();

	// Redirect stdout and stderr
	FILE* fp;
	freopen_s(&fp, "CONOUT$", "w", stdout);
	freopen_s(&fp, "CONOUT$", "w", stderr);
	
	// Find DirectX 11 DLL
	HANDLE processHandle = GetModuleHandle(L"d3d11");
	if (!processHandle) return false;
```

# Getting address of IDXGISwapChain::Present
To redirect the function, I first had to know its location in memory (in the code below, stored in `fnIDXGISwapChainPresent`). This could be done by creating a dummy DirectX device and then walking the VMT from the address of the swap chain.

```
	// Basic swap chain description
	DXGI_SWAP_CHAIN_DESC sd{ 0 };
	sd.BufferCount = 1;
	sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
	sd.BufferDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
	sd.OutputWindow = GetDesktopWindow();
	sd.Windowed = TRUE;
	sd.SwapEffect = DXGI_SWAP_EFFECT_DISCARD;
	sd.SampleDesc.Count = 1;

	// Create device and swap chain
	ID3D11Device* pDummyDevice = nullptr;
	IDXGISwapChain* pDummySwapchain = nullptr;
	HRESULT hr = D3D11CreateDeviceAndSwapChain(nullptr, D3D_DRIVER_TYPE_HARDWARE, nullptr, 0, nullptr, 0, D3D11_SDK_VERSION, &sd, &pDummySwapchain, &pDummyDevice, nullptr, nullptr);
	if (FAILED(hr))
	{
		// Error checking...
	}

	// Calculate address of IDXGISwapChain::Present from address of swapchain
	void** pVMT = *(void***)pDummySwapchain;
	fnIDXGISwapChainPresent = (IDXGISwapChainPresent)(pVMT[8]); // 8 is the magic number

	// Clean up DirectX
	pDummyDevice->Release();
	pDummySwapchain->Release();
```

# Trampoline hooking IDXGISwapChain::Present
Hooking works by dynamically intercepting function calls and replacing a few instructions with a jump to our own function. This can get pretty low level, but thankfully Microsoft has had a [Detours API](https://www.microsoft.com/en-us/research/project/detours/) since 2002, that is still actively maintained and works with Windows 10. This is best illustrated like this:
![Trampoline diagram](https://raw.githubusercontent.com/wiki/microsoft/Detours/Images/Detours_ovr_Technical_Figure_2.png)

Thankfully, this made it quite easy to make my own Present function:

```
	DetourTransactionBegin();
	DetourUpdateThread(GetCurrentThread());
	DetourAttach(&(LPVOID&)fnIDXGISwapChainPresent, (PBYTE)IDXGISwapChain_Present);
	DetourTransactionCommit();
	
	// Keep DLL alive
	while (true)
	{
		
	}

	// Clean up console and thread
	fclose(fp);
	FreeConsole();
	FreeLibraryAndExitThread(hModule, 0);
	return 0;
}
```

The custom function could grab the pointer to the swap chain, then initialise Direct X and what-not if necessary, then begin rendering.

```
HRESULT __stdcall IDXGISwapChain_Present(IDXGISwapChain* pChain, UINT SyncInterval, UINT Flags)
{
	if (!bGraphicsInitialised)
	{
		// Init Direct X
		if (!InitDirectXAndInput(pChain)) return fnIDXGISwapChainPresent(pChain, SyncInterval, Flags);
		
		// Init ImGui
		...
		
		bGraphicsInitialised = true;
	}
	
	// Render ImGui and hand back control
	...
	
}
```

# Direct X initialisation
Getting the device, device context and window from the swap chain was surprisingly easy.

```
bool InitDirectXAndInput(IDXGISwapChain* pChain)
{
	// Get swap chain
	pSwapchain = pChain;

	// Get device and context
	HRESULT hr = pSwapchain->GetDevice(__uuidof(ID3D11Device), (PVOID*)&pDevice);
	if (FAILED(hr))
	{
		std::cerr << "Failed to get device from swapchain" << std::endl;
		return false;
	}
	pDevice->GetImmediateContext(&pContext);

	// Get window from swap chain description
	DXGI_SWAP_CHAIN_DESC swapchainDescription;
	pSwapchain->GetDesc(&swapchainDescription);
	hWindow = swapchainDescription.OutputWindow;

	// Use SetWindowLongPtr to modify window behaviour and get input
	wndProcHandlerOriginal = (WNDPROC)SetWindowLongPtr(hWindow, GWLP_WNDPROC, (LONG_PTR)hWndProc);
	
	// Print resolution as a test
	std::cout << "Successfully initialised DirectX - resolution " 
			<< swapchainDescription.BufferDesc.Width << "x" 
			<< swapchainDescription.BufferDesc.Height << std::endl;
	return true;
}
```

The most interesting part was with `hWindow = swapchainDescription.OutputWindow` and `(WNDPROC)SetWindowLongPtr(hWindow, GWLP_WNDPROC, (LONG_PTR)hWndProc)`, where I got the window and used it to modify its behaviour, allowing me to process input in my own callback for ImGui.

```
LRESULT CALLBACK hWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
	// Process input and pass to ImGui
	if (ImGui_ImplWin32_WndProcHandler(hWnd, uMsg, wParam, lParam))
		return true;

	return CallWindowProc(wndProcHandlerOriginal, hWnd, uMsg, wParam, lParam);
}
```

# Setting up ImGui
To draw, ImGui needed a reference to the window, the device, the device context, as well as an `ID3D11RenderTargetView`, which could be created by the device, assuming a back buffer was present.

```
// Init ImGui
ImGui::CreateContext();
ImGuiIO& io = ImGui::GetIO(); (void)io;
io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;
		
// Setup rendering and IO
ImGui_ImplWin32_Init(hWindow);
ImGui_ImplDX11_Init(pDevice, pContext);
ImGui::GetIO().ImeWindowHandle = hWindow;

// Create render target view for rendering
ID3D11Texture2D* pBackBuffer;
pChain->GetBuffer(0, __uuidof(ID3D11Texture2D), (LPVOID*)&pBackBuffer);
HRESULT hr = pDevice->CreateRenderTargetView(pBackBuffer, NULL, &pRenderTargetView);
if (FAILED(hr))
{ 
	std::cerr << "Failed to create render target view" << std::endl; 
	return fnIDXGISwapChainPresent(pChain, SyncInterval, Flags);
}
pBackBuffer->Release();
```

# Finally rendering
Now all that was left to do was, in my own `IDXGISwapChain_Present` function, render ImGui every frame with my own `ID3D11RenderTargetView`, before passing back control to the actual `IDXGISwapChain::Present`.

```
// Render ImGui
ImGui_ImplDX11_NewFrame();
ImGui_ImplWin32_NewFrame();

ImGui::NewFrame();
bool bShow = true;
ImGui::ShowDemoWindow(&bShow);
ImGui::EndFrame();

ImGui::Render();

pContext->OMSetRenderTargets(1, &pRenderTargetView, NULL);
ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());

return fnIDXGISwapChainPresent(pChain, SyncInterval, Flags);
```

I just had to inject the DLL into Saints Row the Third and everything worked nicely.

![Injecting the DLL](/img/dx11-hooking/Injection.png)

# Conclusion
Creating an overlay in a DirectX 11 application isn't especially difficult, and will work across  the majority of titles. As long as you can hook the `IDXGISwapChain::Present` function and extract the swap chain, device, etc, you can do pretty much whatever you want. Future ideas could include:
* Extracting the projection and view matrices to render 3D models in the game world.
* Hooking other functions such as `ID3D11Device::CreatePixelShader` to modify game shaders and create interesting effects.