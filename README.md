Hello,

I would like to propose a new mod for the official Windhawk catalog.

**Problem:**
In Windows 11 (and recent Windows 10 versions), when using the "Save As" dialog, accidentally hovering the mouse cursor over an existing file automatically overwrites the text box with that file's name. This causes frequent frustrations during quick typing or sequential saving.

**Solution:**
This mod sub-classes `SysListView32` and `ItemsView` to block the `LVN_HOTTRACK` notification triggered by mouse hover. However, it preserves `LVN_ITEMCHANGED`, meaning users can still intentionally click on a file to use/modify its name (which is perfect for sequential file saving like file_v1, file_v2, etc.).

I have thoroughly tested this mod in daily use and it works flawlessly.

Below is the source code for the mod. Thank you for your amazing work on Windhawk!

// ==WindhawkMod==
// @id              fix-saveas-hover-v2
// @name            Fix Save As Mouse Hover - Selective Click
// @description     Ignores accidental mouse hover file selection in Save As dialogs but allows intentional selection via a single click.
// @version         1.3
// @author          raonte
// @github          https://github.com/raonte
// @include         explorer.exe
// @include         *
// ==/WindhawkMod==

#include <windows.h>
#include <shlobj.h>
#include <commctrl.h>

typedef BOOL (WINAPI *SetWindowSubclass_t)(HWND, SUBCLASSPROC, UINT_PTR, DWORD_PTR);
typedef LRESULT (WINAPI *DefSubclassProc_t)(HWND, UINT, WPARAM, LPARAM);

SetWindowSubclass_t pSetWindowSubclass = NULL;
DefSubclassProc_t pDefSubclassProc = NULL;

LRESULT CALLBACK NewListViewProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam, UINT_PTR uIdSubclass, DWORD_PTR dwRefData) {
    if (uMsg == WM_NOTIFY) {
        LPNMHDR lpnmh = (LPNMHDR)lParam;
        if (lpnmh) {
            // 1. Blocchiamo il passaggio del mouse involontario (Hot Track)
            if (lpnmh->code == LVN_HOTTRACK) {
                return 1; 
            }
            // 2. Lasciamo passare le notifiche di selezione reale (clic del mouse o freccia tastiera)
            if (lpnmh->code == LVN_ITEMCHANGED) {
                NMLISTVIEW* pnmv = (NMLISTVIEW*)lParam;
                if (pnmv && (pnmv->uNewState & LVIS_SELECTED)) {
                    // Lasciamo che Windows gestisca il click normalmente
                }
            }
        }
    }
    if (pDefSubclassProc) {
        return pDefSubclassProc(hWnd, uMsg, wParam, lParam);
    }
    return DefWindowProc(hWnd, uMsg, wParam, lParam);
}

using CreateWindowExW_t = decltype(&CreateWindowExW);
CreateWindowExW_t CreateWindowExW_Original;

HWND WINAPI CreateWindowExW_Hook(
    DWORD dwExStyle, LPCWSTR lpClassName, LPCWSTR lpWindowName, DWORD dwStyle,
    int X, int Y, int nWidth, int nHeight, HWND hWndParent, HMENU hMenu,
    HINSTANCE hInstance, LPVOID lpParam) {
    
    HWND hWnd = CreateWindowExW_Original(dwExStyle, lpClassName, lpWindowName, dwStyle, X, Y, nWidth, nHeight, hWndParent, hMenu, hInstance, lpParam);

    if (hWnd && lpClassName && (ULONG_PTR)lpClassName > 0xFFFF) {
        if (wcscmp(lpClassName, L"SysListView32") == 0 || wcscmp(lpClassName, L"ItemsView") == 0) {
            if (pSetWindowSubclass) {
                pSetWindowSubclass(hWnd, NewListViewProc, 1, 0);
            }
        }
    }
    return hWnd;
}

BOOL Wh_ModInit() {
    HMODULE hComCtl = GetModuleHandle(L"comctl32.dll");
    if (!hComCtl) hComCtl = LoadLibrary(L"comctl32.dll");

    if (hComCtl) {
        pSetWindowSubclass = (SetWindowSubclass_t)GetProcAddress(hComCtl, "SetWindowSubclass");
        pDefSubclassProc = (DefSubclassProc_t)GetProcAddress(hComCtl, "DefSubclassProc");
    }

    Wh_SetFunctionHook((void*)CreateWindowExW, (void*)CreateWindowExW_Hook, (void**)&CreateWindowExW_Original);
    return TRUE;
}
