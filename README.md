#include <iostream>
#include <Windows.h>
#include <TlHelp32.h>

DWORD GetProcessIdByName(const char* processName)
{
    PROCESSENTRY32 entry;
    entry.dwSize = sizeof(PROCESSENTRY32);
    const auto snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, NULL);
    if (Process32First(snapshot, &entry) == TRUE)
    {
        while (Process32Next(snapshot, &entry) == TRUE)
        {
            if (strcmp(entry.szExeFile, processName) == 0)
            {
                CloseHandle(snapshot);
                return entry.th32ProcessID;
            }
        }
    }
    CloseHandle(snapshot);
    return 0;
}

int main()
{
    const char* processName = "RobloxPlayerBeta.exe";
    const char* dllName = "path/to/your/dll.dll";

    const auto processId = GetProcessIdByName(processName);
    if (!processId)
    {
        std::cout << "Failed to find Roblox process" << std::endl;
        return 1;
    }

    const auto processHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, processId);
    if (!processHandle)
    {
        std::cout << "Failed to open process" << std::endl;
        return 1;
    }

    const auto allocMem = VirtualAllocEx(processHandle, NULL, strlen(dllName) + 1, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
    if (!allocMem)
    {
        std::cout << "Failed to allocate memory" << std::endl;
        return 1;
    }

    if (!WriteProcessMemory(processHandle, allocMem, dllName, strlen(dllName) + 1, NULL))
    {
        std::cout << "Failed to write to process memory" << std::endl;
        return 1;
    }

    const auto loadLibraryAddr = reinterpret_cast<LPTHREAD_START_ROUTINE>(GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA"));
    if (!loadLibraryAddr)
    {
        std::cout << "Failed to get LoadLibraryA address" << std::endl;
        return 1;
    }

    const auto threadHandle = CreateRemoteThread(processHandle, NULL, NULL, loadLibraryAddr, allocMem, NULL, NULL);
    if (!threadHandle)
    {
        std::cout << "Failed to create remote thread" << std::endl;
        return 1;
    }

    std::cout << "Injected " << dllName << " into Roblox" << std::endl;
    CloseHandle(threadHandle);
    CloseHandle(processHandle);
    return 0;
}
