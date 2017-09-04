---
title:  "Code Snippeds"
date:   2017-09-04
categories: C++
tags: [C++]
excerpt: "Some short and useful code snippets."
---

This is a set of some code snippets that you might find useful. 

The 1st one is pretty popular, but I use it in the snippets below. Basically it a way to retrieve the size of a C++ array.

```cpp
template <typename T, size_t N> char (&TArrSize_Safe(T (&)[N]))[N];
#define ARRSZ(A) (sizeof(TArrSize_Safe(A)))

// Usage
int myArray[1234];
printf("myArray has %d elements!" ARRSZ(myArray));
```

This is a printf style **std::string** formatting:  
```cpp
std::string string_format(const char* const fmt_str, ...)
{
	va_list ap;
	va_start(ap, fmt_str);
	const size_t ln = vsnprintf(nullptr, 0, fmt_str, ap);
	va_end(ap);

	std::string retval(ln, 'a');

	va_start(ap, fmt_str);
	snprintf(&retval[0], retval.size()+1, fmt_str, ap);
	va_end(ap);
}
```
  
  
This one is a simple file open dialog. WINAPI is used for Windows, and zenity for GNU/Linux:
```cpp
template <typename T, size_t N> char (&TArrSize_Safe(T (&)[N]))[N];
#define ARRSZ(A) (sizeof(TArrSize_Safe(A)))


std::string FileOpenDialog(const std::string& prompt, bool fileMustExists, const char* fileFilter)
{
#ifdef WIN32

	static std::mutex mtx;
	std::lock_guard<std::mutex> mtx_guard(mtx);

	const int BUFSIZE = 1024;
	char buffer[BUFSIZE] = {0};

	TCHAR currentDir[MAX_PATH];

	// GetOpenFileName changes the current directory. We do not want that so we revert it back to what it was.
	if(GetCurrentDirectory(ARRAYSIZE(currentDir), currentDir)!=0)
	{
		OPENFILENAME ofns = {0};
		ofns.lStructSize = sizeof(ofns);
		ofns.lpstrFile = buffer;
		ofns.nMaxFile = BUFSIZE;
		ofns.lpstrTitle = prompt.c_str();
		ofns.lpstrFilter = fileFilter ? fileFilter : "All Files\0*.*\0";
		
		if(fileMustExists)
		{
			ofns.Flags |= OFN_FILEMUSTEXIST;
		}

		const BOOL okClicked  = GetOpenFileNameA(&ofns);

		SetCurrentDirectory(currentDir);
	}
	else
	{
		assert(false); // Should never happen!
	}

	return std::string(buffer);
#else
	char file[1024] = {0};
	FILE* const f = popen("zenity --file-selection", "r"); // [TODO} File extension filters.
	assert(f!=nullptr);
	fgets(file, 1024, f);
	file[ARRSZ(file)] = '\0'; // Clamp if the filename is too long.
	std::string s(file); // [TODO] Zenitiy.
	s.pop_back(); // Delete the '\n' printed by zenity.
	pclose(f);
	return s;
#endif
}
```