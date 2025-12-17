# GLAD (OpenGL & WGL Loader)

이 저장소는 OpenGL 및 WGL(Windows Graphics Library) 확장을 로드하기 위한 GLAD 소스 코드를 포함하고 있습니다.
별도의 라이브러리로 빌드되지 않으며, 상위 프로젝트에서 **서브모듈(Submodule)**로 추가하여 소스 코드를 직접 포함해 빌드하도록 설계되었습니다.

## 📂 디렉토리 구조

```text
glad/
├── include/
│   ├── glad/
│   │   ├── glad.h        # OpenGL 헤더 (Core/Compatibility)
│   │   └── glad_wgl.h    # WGL 확장 헤더 (Windows 전용)
│   └── KHR/
│       └── khrplatform.h # 플랫폼 호환성 헤더
└── src/
    ├── glad.c            # OpenGL 로더 구현부
    └── glad_wgl.c        # WGL 로더 구현부

```

## 🛠️ 통합 방법 (Integration)

이 프로젝트를 서브모듈로 추가한 뒤, 빌드 시스템(예: CMake)에 소스 파일과 인클루드 경로를 등록하여 사용합니다.

### 1. 서브모듈 추가

```bash
git submodule add <이 저장소의 URL> glad

```

### 2. CMakeLists.txt 설정 예시

메인 프로젝트의 `CMakeLists.txt`에서 `glad.c`와 `glad_wgl.c`를 컴파일 목록에 추가하고, `include` 디렉토리를 참조하도록 설정합니다.

```cmake
# GLAD 경로 설정 (서브모듈 경로)
set(GLAD_DIR "${CMAKE_CURRENT_SOURCE_DIR}/glad")

# 1. 소스 파일 목록 지정 (OpenGL + WGL 로더)
set(GLAD_SOURCES
    ${GLAD_DIR}/src/glad.c
    ${GLAD_DIR}/src/glad_wgl.c
)

# 2. 실행 파일(또는 라이브러리)에 소스 및 인클루드 경로 추가
add_executable(MyEngine main.cpp ${GLAD_SOURCES})

target_include_directories(MyEngine PRIVATE
    ${GLAD_DIR}/include
)

# 3. Windows 시스템 라이브러리 링크 (opengl32.lib 필수)
if(WIN32)
    target_link_libraries(MyEngine opengl32)
endif()

```

## 💻 사용 예제 (Pure WGL Usage)

GLFW 같은 외부 라이브러리 없이, **Win32 API**를 직접 사용하여 윈도우를 생성하고 GLAD를 초기화하는 예제입니다.

```cpp
#include <windows.h>
#include <iostream>

// GLAD 헤더 (반드시 다른 GL 헤더나 windows.h보다 뒤에, 혹은 매크로 처리 필요할 수 있음)
// 하지만 glad.h는 내부적으로 windows.h가 포함되지 않으므로, windows.h를 먼저 포함해야 합니다.
#include <glad/glad.h>
#include <glad/glad_wgl.h>

LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam);

int main() {
    HINSTANCE hInstance = GetModuleHandle(NULL);
    const char* className = "MyOpenGLWindow";

    // 1. 윈도우 클래스 등록
    WNDCLASS wc = {0};
    wc.lpfnWndProc = WndProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = className;
    wc.style = CS_OWNDC; // OpenGL 컨텍스트를 위해 필수
    RegisterClass(&wc);

    // 2. 윈도우 생성
    HWND hWnd = CreateWindow(
        className, "Pure WGL Window", WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, CW_USEDEFAULT, 800, 600,
        NULL, NULL, hInstance, NULL
    );

    if (!hWnd) return -1;

    // 3. Device Context(HDC) 가져오기
    HDC hdc = GetDC(hWnd);

    // 4. 픽셀 포맷 설정 (Pixel Format Descriptor)
    PIXELFORMATDESCRIPTOR pfd = {
        sizeof(PIXELFORMATDESCRIPTOR), 1,
        PFD_DRAW_TO_WINDOW | PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER, // 더블 버퍼링 필수
        PFD_TYPE_RGBA, 32, 
        0, 0, 0, 0, 0, 0,
        0, 0, 0, 0, 0, 0, 0,
        24, 8, 0, PFD_MAIN_PLANE, 0, 0, 0, 0
    };

    int format = ChoosePixelFormat(hdc, &pfd);
    SetPixelFormat(hdc, format, &pfd);

    // 5. 임시 OpenGL 컨텍스트 생성 (WGL 확장을 로드하기 위함)
    HGLRC tempContext = wglCreateContext(hdc);
    wglMakeCurrent(hdc, tempContext);

    // 6. GLAD WGL 로더 초기화 (wglCreateContextAttribsARB 등을 로드)
    if (!gladLoadWGL(hdc)) {
        std::cout << "Failed to initialize WGL extensions!" << std::endl;
        return -1;
    }

    // 7. (선택 사항) Core Profile 컨텍스트로 업그레이드
    // WGL 확장이 로드되었으므로 wglCreateContextAttribsARB 사용 가능
    // 여기서는 간단히 기존 컨텍스트를 계속 사용하거나, 
    // 실제 엔진에서는 여기서 새 컨텍스트를 만들고 tempContext를 삭제하는 것이 일반적입니다.

    // 8. GLAD OpenGL 로더 초기화 (glDrawArrays 등 로드)
    if (!gladLoadGL()) {
        std::cout << "Failed to initialize OpenGL!" << std::endl;
        return -1;
    }

    ShowWindow(hWnd, SW_SHOW);

    // 9. 메인 루프
    MSG msg = {0};
    while (true) {
        if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE)) {
            if (msg.message == WM_QUIT) break;
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        } else {
            // 렌더링 코드
            glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
            glClear(GL_COLOR_BUFFER_BIT);

            SwapBuffers(hdc); // 화면 갱신
        }
    }

    // 종료 처리
    wglMakeCurrent(NULL, NULL);
    wglDeleteContext(tempContext);
    ReleaseDC(hWnd, hdc);
    DestroyWindow(hWnd);

    return 0;
}

LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam) {
    switch (message) {
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    }
    return DefWindowProc(hWnd, message, wParam, lParam);
}

```

## ⚠️ 주의사항

* **헤더 포함 순서**: `<windows.h>`를 `<glad/glad.h>`보다 **먼저** 포함해야 합니다. (혹은 `APIENTRY` 관련 매크로 충돌 방지가 필요할 수 있음)
* **WGL 의존성**: 이 코드는 Windows 전용입니다. `glad_wgl.c`와 `glad_wgl.h`는 Linux나 macOS에서는 컴파일되지 않으므로, 크로스 플랫폼 빌드시 `CMakeLists.txt`에서 `if(WIN32)` 분기 처리가 필요합니다.
* **Context Upgrade**: 위 예제는 간단한 설명을 위해 레거시 컨텍스트 생성(`wglCreateContext`)을 보여줍니다. OpenGL 3.3+ Core Profile을 사용하려면 6번 단계 이후 `wglCreateContextAttribsARB`를 사용하여 새 컨텍스트를 생성해야 합니다.