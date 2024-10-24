# Youpk
Another active de-sheller based on ART



## Principle

Youpk is a de-sheller for Dex overall reinforcement + various Dex extraction

The basic process is as follows:

1. Dump DEX from memory
2. Construct a complete call chain, actively call all methods and dump CodeItem
3. Merge DEX, CodeItem

### Dump DEX from memory

DEX files are represented by DexFile objects in the art virtual machine, and these objects are referenced in ClassLinker, so they can be obtained by traversing DexFile objects from ClassLinker and dumping them.

```c++
//unpacker.cc
std::list<const DexFile*> Unpacker::getDexFiles() {
  std::list<const DexFile*> dex_files;
  Thread* const self = Thread::Current();
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  ReaderMutexLock mu(self, *class_linker->DexLock());
  const std::list<ClassLinker::DexCacheData>& dex_caches = class_linker->GetDexCachesData();
  for (auto it = dex_caches.begin(); it != dex_caches.end(); ++it) {
    ClassLinker::DexCacheData data = *it;
    const DexFile* dex_file = data.dex_file;
    dex_files.push_back(dex_file);
  }
  return dex_files;
}
```

In addition, to avoid any form of optimization in dex affecting the dumped dex file, set the CompilerFilter in dex2oat to only verify

```c++
//dex2oat.cc
compiler_options_->SetCompilerFilter(CompilerFilter::kVerifyAtRuntime);
```



### Construct a complete call chain and actively call all methods

1. Create an unpacking thread

   ```java
   //unpacker.java
   public static void unpack() {
       if (Unpacker.unpackerThread != null) {
           return;
       }
   
       //Start thread call
       Unpacker.unpackerThread = new Thread() {
           @Override public void run() {
               while (true) {
                   try {
                       Thread.sleep(UNPACK_INTERVAL);
                   }
                   catch (InterruptedException e) {
                       e.printStackTrace();
                   }
                   if (shouldUnpack()) {
                       Unpacker.unpackNative();
                   }   
               }
           }
       };
       Unpacker.unpackerThread.start();
   }
   ```

2. Traverse all ClassDefs of DexFile in the unpacking thread

   ```c++
   //unpacker.cc
   for (; class_idx < dex_file->NumClassDefs(); class_idx++) {
   ```

3. Parse and initialize Class

   ```c++
   //unpacker.cc
   mirror::Class* klass = class_linker->ResolveType(*dex_file, dex_file->GetClassDef(class_idx).class_idx_, h_dex_cache, h_class_loader);
   StackHandleScope<1> hs2(self);
   Handle<mirror::Class> h_class(hs2.NewHandle(klass));
   bool suc = class_linker->EnsureInitialized(self, h_class, true, true);
   ```

4. Actively call all methods of the Class, and modify ArtMethod::Invoke to force it to use the switch interpreter

   ```c++
   //unpacker.cc
   uint32_t args_size = (uint32_t)ArtMethod::NumArgRegisters(method->GetShorty());
   if (!method->IsStatic()) {
       args_size += 1;
   }
   
   JValue result;
   std::vector<uint32_t> args(args_size, 0);
   if (!method->IsStatic()) {
       mirror::Object* thiz = klass->AllocObject(self);
       args[0] = StackReference<mirror::Object>::FromMirrorPtr(thiz).AsVRegValue();  
   }
   method->Invoke(self, args.data(), args_size, &result, method->GetShorty());
   
   //art_method.cc
   if (UNLIKELY(!runtime->IsStarted() || Dbg::IsForcedInterpreterNeededForCalling(self, this) 
   || (Unpacker::isFakeInvoke(self, this) && !this->IsNative()))) {
   if (IsStatic()) {
   art::interpreter::EnterInterpreterFromInvoke(
   self, this, nullptr, args, result, /*stay_in_interpreter*/ true);
   } else {
   mirror::Object* receiver =
   reinterpret_cast<StackReference<mirror::Object>*>(&args[0])->AsMirrorPtr();
   art::interpreter::EnterInterpreterFromInvoke(
   self, this, receiver, args + 1, result, /*stay_in_interpreter*/ true);
   }
   }
   
   //interpreter.cc
   static constexpr InterpreterImplKind kInterpreterImplKind = kSwitchImplKind;
   ```

5. Insert stubs in the interpreter and set callbacks before each instruction is executed

   ```c++
   //interpreter_switch_impl.cc
   // Code to run before each dex instruction.
     #define PREAMBLE()                                                                 \
     do {                                                                               \
       inst_count++;                                                                    \
       bool dumped = Unpacker::beforeInstructionExecute(self, shadow_frame.GetMethod(), \
                                                        dex_pc, inst_count);            \
       if (dumped) {                                                                    \
         return JValue();                                                               \
       }                                                                                \
       if (UNLIKELY(instrumentation->HasDexPcListeners())) {                            \
         instrumentation->DexPcMovedEvent(self, shadow_frame.GetThisObject(code_item->ins_size_),  shadow_frame.GetMethod(), dex_pc);            						   										   \
       }                                                                                \
     } while (false)
   ```

6. Dump the targeted CodeItem in the callback. Here is just a simple example of direct dumping. In fact, for some manufacturers’ extraction, you can actually execute a few instructions and wait for the CodeItem to be decrypted before dumping.

   ```c++
   //unpacker.cc
   bool Unpacker::beforeInstructionExecute(Thread *self, ArtMethod *method, uint32_t dex_pc, int inst_count) {
     if (Unpacker::isFakeInvoke(self, method)) {
     	Unpacker::dumpMethod(method);
       return true;
     }
     return false;
   }
   ```



### Merge DEX, CodeItem

Just fill the dumped CodeItem into the corresponding position of DEX. Mainly based on the modification of google dx tool.



### Reference Links

FUPK3: https://bbs.pediy.com/thread-246117.htm

FART: https://bbs.pediy.com/thread-252630.htm



## Flash the device

1. Only supports pixel 1 generation
2. Reboot to bootloader: `adb reboot bootloader`
3. Unzip Youpk_sailfish.zip and double-click `flash-all.bat`



## Compile

### Sheller source code compilation

1. Download the complete source code of android-7.1.2_r33
2. Replace unpacker/android-7.1.2_r33
3. Compile

### Repair tool compilation

1. Import dexfixer project into IDEA
2. main class为 `com.android.dx.unpacker.DexFixer` 



## Usage

1. **This tool is only used for learning and communication. Please do not use it for illegal purposes, otherwise you will bear the consequences yourself! **
   
2. Configure the app package name to be unpacked, or more precisely, the process name

    ```bash
    adb shell "echo cn.youlor.mydemo >> /data/local/tmp/sunlake.config"
    ```

3. If the apk is not fully reinforced and installd does not call dex2oat for optimization, you need to execute step 2 before installation
    
4. Start apk and wait for unpacking
    The decompression will be automatically re-dumped every 10 seconds (the dex that has been completely dumped will be ignored). The decompression is completed when the log prints "unpack end"

5. Pull out the dump file, the dump file path is `/data/data/package name/sunlake`

    ```bash
    adb pull /data/data/cn.youlor.mydemo/sunlake
    ```

6. Call the repair tool dexfixer.jar, two parameters, the first is the dump file directory (must be a valid path), the second is the reorganized DEX directory (will be created if it does not exist)
    ```bash
    java -jar dexfixer.jar /path/to/sunlake /path/to/output
    ```



## Applicable scenarios

1. Overall reinforcement
2. Extraction:
   - nop pit type (similar to some encryption)
   - Naitve, decrypted in `<clinit>` (similar to early Alibaba)
   - goto decryption type (similar to the new version of a certain encryption, najia): https://bbs.pediy.com/thread-259448.htm




## Frequently Asked Questions

1. If the dump exits or gets stuck midway, restart the process and wait for the dump to be unpacked again.
2. Currently only supports dex protected by shell, not dex/jar dynamically loaded by App
