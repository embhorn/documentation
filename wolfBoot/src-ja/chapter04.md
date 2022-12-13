

# ハードウェア抽象化レイヤー



ターゲットマイクロコントローラーでwolfBootを実行するには、HALの実装を提供する必要があります。


HALの目的は、ブートローダーからの書き込み/消去操作と、アプリケーションライブラリを介してファームウェアのアップグレードを開始するアプリケーションを許可し、ブート中にMCUがフルスピードで実行されるようにすることです(署名の検証を最適化するため)。


各プラットフォームのハードウェア固有の呼び出しの実装は、`hal`ディレクトリの単一のCファイルにグループ化されます。


ディレクトリには、サポートされている各MCUのプラットフォーム固有のリンカースクリプトも含まれており、同じ名前と`.ld`拡張機能があります。これは、特定のハードウェアにブートローダーのファームウェアをリンクし、フラッシュ境界とRAM境界に必要なすべてのシンボルをエクスポートするために使用されます。



## サポートされているプラットフォーム



現在のバージョンでは、次のプラットフォームがサポートされています。 
- STM32F4、STM32L5、STM32L0、STM32F7、STM32H7、STM32G0 
- NRF52
- ATMEL SAMR21
- TI CC26X2 
- KINETIS 
- SiFive HiFive1 RISC-V



## API



ハードウェア抽象化レイヤー(HAL)は、サポートされているターゲットごとに6つの関数呼び出しで構成されています。

```
void hal_init(void)
```

この関数は、実行の最初にブートローダーによって呼び出されます。理想的には、提供された実装は、ターゲットマイクロコントローラーのクロック設定を構成し、暗号化プリミティブがファームウェアイメージを確認するために必要な時間を短縮するために必要な速度で実行されるようにします。


```
void hal_flash_unlock(void)
```


ターゲットのフラッシュメモリのIAPインターフェースがそれを必要とする場合、この関数はすべての書き込みおよび消去操作の前に呼び出され、フラッシュへの書き込みアクセスを解除します。一部のターゲットでは、この関数が空になる場合があります。


```
int hal_flash_write(uint32_t address, const uint8_t *data, int len)
```


この関数は、ターゲットのIAPインターフェースを使用して、フラッシュ書き込み関数の実装を提供します。`address`はフラッシュ領域の先頭からのオフセットであり、`data`はIAPインターフェースを使用してフラッシュに保存するペイロード、`len`はペイロードのサイズです。`hal_flash_write`は、成功すると0を返す必要があります。


```
void hal_flash_lock(void)
```


フラッシュメモリのIAPインターフェースにロック/ロック解除が必要な場合、この関数は、書き込みアクセスを除外してフラッシュ書き込み保護を復元します。この関数は、すべての書き込みおよび消去操作の最後にブートローダーによって呼び出されます。

```
int hal_flash_erase(uint32_t address, int len)
```


ブートローダーによって呼び出されて、フラッシュメモリの一部を消去して、後続のブートを許可します。ターゲットマイクロコントローラーの特定のIAPインターフェースを介して、消去操作を実行する必要があります。`address`ブートローダーが消去したいエリアの開始をマークし、`len`は消去するエリアのサイズを指定します。この関数は、フラッシュセクターのジオメトリを考慮し、その間のすべてのセクターを消去する必要があります。


```
void hal_prepare_boot(void)
```


この関数は、次の段階でファームウェアをチェーンでロードする前に、非常に遅い段階でブートローダーによって呼び出されます。これを使用して、マイクロコントローラーの状態が元の設定に復元されるように、クロック設定に行われたすべての変更を戻すことができます。



### 外部フラッシュメモリのオプションのサポート



wolfBootはmakefileコマンドへのオプション`EXT_FLASH=1`でコンパイルできます。外部フラッシュサポートが有効になっている場合、パーティションを更新およびスワップすることが外部メモリに関連付けられ、読み取り/書き込み/消去アクセスに代替HAL機能を使用します。更新またはスワップパーティションを外部メモリに関連付けるには、それぞれ`PART_UPDATE_EXT`および/または`PART_SWAP_EXT`を定義します。


以下の関数は、外部メモリにアクセスするために使用され、`EXT_FLASH`がオンになっている場合に定義する必要があります。


```
int  ext_flash_write(uintptr_t address, const uint8_t *data, int len)
```


この関数は、外部メモリの特定のインターフェースを使用して、フラッシュ書き込み関数の実装を提供します。`address`は、デバイス内のアドレス指定可能なスペースの先頭からのオフセット、`data`は保存するペイロード、`len`はペイロードのサイズです。`ext_flash_write`は、成功すると0を返す必要があります。


```
int  ext_flash_read(uintptr_t address, uint8_t *data, int len)
```


この関数は、ドライバーの特定のインターフェースを使用して、外部メモリの間接的な読み取りを提供します。`address`は、デバイス内のアドレス指定可能なスペースの先頭からのオフセットであり、`data`はコールの成功にペイロードが保存されるポインターであり、`len`はペイロードに許容される最大サイズです。`ext_flash_read`は、成功すると0を返す必要があります。


```
int  ext_flash_erase(uintptr_t address, int len)
```


ブートローダによって呼び出され、外部メモリの一部を消去します。消去操作は、ターゲットドライバーの特定のインターフェース(SPIフラッシュなど)を介して実行する必要があります。`address`は、デバイスに対するエリアの開始をマークし、ブートローダーが消去したいと考え、`len`は消去するエリアのサイズを指定します。この関数は、セクターのジオメトリを考慮し、その間のすべてのセクターを消去する必要があります。


```
void ext_flash_lock(void)
```


外部フラッシュメモリのインターフェースにロック/ロック解除が必要な場合、この関数を使用してフラッシュ書き込み保護を復元するか、書き込みアクセスを除外することができます。この関数は、外部デバイスのすべての書き込みおよび消去操作の最後にブートローダーによって呼び出されます。




```
void ext_flash_unlock(void)
```

外部メモリのIAPインターフェースがそれを必要とする場合、この関数は、すべての書き込みおよび消去操作の前に呼び出され、デバイスへの書き込みアクセスを解除します。一部のドライバーでは、この機能が空になる場合があります。