---
title:  "Code Snippets"
date:   2017-09-04
categories: C++
tags: [C++]
excerpt: "Some short and useful code snippets."
---

This is a set of some code snippets that you might find useful. 

The 1st one is pretty popular, but I use it in the snippets below. Basically it is 

# A way to retrieve the size of a C++ array.

```cpp
template <typename T, size_t N> char (&TArrSize_Safe(T (&)[N]))[N];
#define ARRSZ(A) (sizeof(TArrSize_Safe(A)))

// Usage
int myArray[1234];
printf("myArray has %d elements!" ARRSZ(myArray));
```

# ```printf``` style ```std::string``` formatting:  
```cpp
// The caller is EXPECTED to call va_end on the va_list args
inline void string_format(std::string& retval, const char* const fmt_str, va_list args)
{	
	// [CAUTION]
	// Under Windows with msvc it is fine to call va_start once and then use the va_list multiple times.
	// However this is not the case on the other platforms. The POSIX docs state that the va_list is undefined
	// after calling vsnprintf with it:
	//
	// From https://linux.die.net/man/3/vsprintf :
	// The functions vprintf(), vfprintf(), vsprintf(), vsnprintf() are equivalent to the functions 
	// printf(), fprintf(), sprintf(), snprintf(), respectively, 
	// except that they are called with a va_list instead of a variable number of arguments. 
	// These functions do not call the va_end macro. Because they invoke the va_arg macro, 
	// the value of ap is undefined after the call. See stdarg(3). 
	// Obtain the length of the result string.
	va_list args_copy;
	va_copy(args_copy, args);
	
	const size_t ln = vsnprintf(nullptr, 0, fmt_str, args);

	// [CAUTION] Write the data do the result. Allocate one more element 
	// as the *sprintf function add the '\0' always. Later we will pop that element.
	retval.resize(ln + 1, 'a');

	vsnprintf(&retval[0], retval.size()+1, fmt_str, args_copy);
	retval.pop_back(); // remove the '\0' that was added by snprintf on the back.
	va_end(args_copy);
}

inline void string_format(std::string& retval, const char* const fmt_str, ...)
{
	va_list args;
	va_start(args, fmt_str);
	string_format(retval, fmt_str, args);
	va_end(args);
}

inline std::string string_format(const char* const fmt_str, ...)
{
	std::string retval;
	
	va_list args;
	va_start(args, fmt_str);
	string_format(retval, fmt_str, args);
	va_end(args);

	return retval;
}
```

I guess it could be done in a safer way by using a ```char*``` or ```std::unique_ptr<char[]>``` to get the result of ```vsnprintf``` and then transfer it to ```std::string```,
 but that one works just fine.

# Simple file open dialog.

WINAPI is used for Windows, and zenity for GNU/Linux:
```cpp
template <typename T, size_t N> char (&TArrSize_Safe(T (&)[N]))[N];
#define ARRSZ(A) (sizeof(TArrSize_Safe(A)))


std::string FileOpenDialog(const std::string& prompt)
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
		ofns.Flags |= OFN_FILEMUSTEXIST;
		
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
	FILE* const f = popen("zenity --file-selection", "r");
	assert(f!=nullptr);
	fgets(file, 1024, f);
	file[ARRSZ(file)] = '\0'; // Clamp if the filename is too long.
	std::string s(file);
	
	// Usually there is a newline symbol, if so remove it.
	if(s.size() > 0 && s.back() == '\n') {
		s.pop_back();
	}
	
	pclose(f);
	return s;
#endif
}
```
---
# A ```std::chrono```  based timer:

```cpp
struct Timer
{
	typedef std::chrono::high_resolution_clock clock;
	typedef clock::time_point time_point;

	// The one below is set in a *.cpp to this:
	// const Timer::time_point Timer::application_start_time = Timer::clock::now();
	static const time_point application_start_time;

	Timer()
	{
		reset();
	}

	void reset()
	{
		lastUpdate = clock::now();
		dtSeconds = 0.f;
	}

	// Returns the current time since the application start in seconds.
	static float now_seconds()
	{
		return std::chrono::duration_cast<std::chrono::microseconds>(clock::now() - application_start_time).count() * 1e-6f;
	}

	void tick()
	{
		const time_point now = clock::now();
		const auto diff = std::chrono::duration_cast<std::chrono::microseconds>(now - lastUpdate).count();
		lastUpdate = now;

		dtSeconds = diff * 1e-6f; // microseconds -> seconds
	}

	float diff_seconds() const { return dtSeconds; }

private :

	time_point lastUpdate;
	float dtSeconds;
};
```