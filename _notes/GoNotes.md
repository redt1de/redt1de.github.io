---
layout: post
title: Random Go Notes
date: 2023-10-13 10:00:00 -500
categories: [Notes, Golang]
tags: [Notes,Golang]
note: true
---


## Building

### Turning off optimization and inlining in Go gc compilers
```bash
go build -gcflags '-N -l' [code.go]
```

### cross compile to EXE,garble,CGO
```bash
GOGARBLE=* GOPRIVATE=* GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc CXX=x86_64-w64-mingw32-g++ go build -a -trimpath -buildvcs=false -ldflags -s -w -buildid= -H=windowsgui "-extldflags=-s -W -w" -o [outfile.exe]
```
### cross compile to EXE,garble
```bash
GOGARBLE=* GOPRIVATE=* GOOS=windows GOARCH=amd64 go build -a -trimpath -buildvcs=false -ldflags -s -w -buildid= -H=windowsgui "-extldflags=-s -W -w" -o [outfile.exe]
```
### cross compile to DLL, garble,CGO
```bash
GOGARBLE=* GOPRIVATE=* GOOS=windows GOARCH=amd64 CGO_ENABLED=1 CC=x86_64-w64-mingw32-gcc CXX=x86_64-w64-mingw32-g++ go build -buildmode=c-shared -a -trimpath -buildvcs=false -ldflags -s -w -buildid= -H=windowsgui "-extldflags=-s -W -w" -o [outfile.dll]
```

## Snippets

### Get a pointer to a function without windows.NewCallback() or funky typecast
```go
fnPtr := reflect.ValueOf(someFunc).Pointer()
```

### Non api/syscall mem read/write (only for self, not remote procs)
```go
// WriteMemory is a non system call mem write func. Does **not** check permissions, may cause panic if memory is not writable etc. https://github.com/timwhitez/Doge-Misc/blob/main/writeMem.go
func WriteMemory(inbuf []byte, destination uintptr) {
	for index := uint32(0); index < uint32(len(inbuf)); index++ {
		writePtr := unsafe.Pointer(destination + uintptr(index))
		v := (*byte)(writePtr)
		*v = inbuf[index]
	}
}

// ReadMemory
func ReadMemory(addr uintptr, readLen int) []byte{
	readmem := unsafe.Slice((*byte)(unsafe.Pointer(addr)), readLen)
    return readmem
}
```

### LPWSTR to string
```go
func lpwstrToString(cwstr C.LPCWSTR) string {
	const maxRunes = 1<<30 - 1
	ptr := unsafe.Pointer(cwstr)
	sz := C.wcslen((*C.wchar_t)(ptr))
	wstr := (*[maxRunes]uint16)(ptr)[:sz:sz]
	return string(utf16.Decode(wstr))
}
```

### UNICODE_STRING 
```go
type UNICODE_STRING struct {
	Length        uint16
	MaximumLength uint16
	Buffer        *uint16
}

func NewUnicodeString(s string) UNICODE_STRING {
	ws, _ := windows.UTF16PtrFromString(s)
	len := uint16(len(s) * 2) // 2 bytes per character in UTF-16
	return UNICODE_STRING{
		Length:        len,
		MaximumLength: len,
		Buffer:        ws,
	}
}
func (u UNICODE_STRING) String() string {
	return windows.UTF16PtrToString(u.Buffer)
}
```                            

### Hexdump a pointer to anything
```go
// HexDump prints a hex dump of the memory at the given pointer. 
// Although type is interface{}, it must be passed a pointer to the object. HexDump(&someVar)
func HexDump(a interface{}) {
	val := reflect.ValueOf(a)
	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}
	size := val.Type().Size()

	addr := val.UnsafeAddr()
	hx := unsafe.Slice((*byte)(unsafe.Pointer(addr)), size)
	println(hex.Dump(hx))
}
```