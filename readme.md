MemoryModule
============

Функции, которые используются для загрузки модулей (LoadLibrary, LoadLibraryEx) не поддерживают загрузку модулей из памяти.
Данный модуль реализует данную функцию.

Основа была взята с репозитария: https://github.com/fancycode/MemoryModule

Вносятся правки которые позволяют полноценно запускать EXE из памяти.

История:

15.02.16 - Реализована возможность указания командной строки перед запуском EntryPoint.
Поток, исполняющийся внутри MemoryModule маркируется изменением значения ProcessParameters в _PEB.
Производится перехват (методом импорта) некоторых важных функций для изоляции от основного процесса.
!Модуль не учитывается в списке модулей процесса!

16.02.16 - Некоторые фиксы в TLS.

19.02.16 - Введена функция TryFreeMem и флаг bForceMemFree для "насильного" освобождения памяти в случае её занятости.
!Использовать, только если основной процесс больше не будет использоваться, иначе могут быть ошибки!

Проблемы:
1. При отсутствии таблицы .reloc и невозможности выделить нужную память по ImageBase запуск модуля становится невозможным. 

Пример запуска:

	wchar_t ModuleName[64+1];
	memset(ModuleName, 0x00, (64+1)*2);

	GetModuleFileNameW(NULL, ModuleName, 64);

	HMODULE hThisModule = GetModuleHandle(L"MyDllOrExe.dll");

	HRSRC hResInfo = FindResource(hThisModule, MAKEINTRESOURCE(IDR_EXE1), RT_RCDATA);
	HGLOBAL hRes = LoadResource(hThisModule, hResInfo);
	char *resData = (char*)LockResource(hRes);

	IMAGE_DOS_HEADER *hDOS = (IMAGE_DOS_HEADER*)resData;
	IMAGE_NT_HEADERS32 *hNT = (IMAGE_NT_HEADERS32*)(resData+hDOS->e_lfanew);

	HMEMORYMODULE hM = MemoryLoadLibrary(resData, hNT->OptionalHeader.SizeOfImage, true);
	hResInfo = FindResource(hThisModule, MAKEINTRESOURCE(IDR_PARAMS1), RT_RCDATA);
	hRes = LoadResource(hThisModule, hResInfo);
	wchar_t *CommandLine = (wchar_t*)LockResource(hRes);

	MEMORYMODULE_START_ARGS moduleStartArgs;	// Startup struct
	moduleStartArgs.EntryPoint = NULL;		// Start func address. If NULL - is EntryPoint
	moduleStartArgs.CommandLine = CommandLine;	// CommandLine
	moduleStartArgs.InThread = true;		// Run in new thread
	MemoryCallEntryPoint(hM, &moduleStartArgs);	// CALL
