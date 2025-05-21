# tts 基本流程

## 方法签名

def tts(
        self,
        text: str = "",
        speaker_name: str = "",
        language_name: str = "",
        speaker_wav=None,
        style_wav=None,
        style_text=None,
        reference_wav=None,
        reference_speaker_name=None,
        split_sentences: bool = True,
        **kwargs,
    ) -> List[int]:
## 参数说明：
text: 要合成的文本。

speaker_name: 说话人名称（适用于多说话人模型）。

language_name: 语言名称（适用于多语种模型）。

speaker_wav: 用于说话人嵌入的音频（语音克隆）。

style_wav: 语音风格参考音频（用于风格迁移）。

style_text: style_wav 对应的文本。

reference_wav: 用于语音转换的参考音频。

reference_speaker_name: 参考音频的说话人名称。

split_sentences: 是否将输入文本分句处理。

**kwargs: 其他传入 TTS 模型的自定义参数。

## 前期准备
```
start_time = time.time()
wavs = []

if not text and not reference_wav:
    raise ValueError(...)
```
开始计时。

如果既没有提供 text 也没有提供 reference_wav，报错：至少要有一个用于合成或语音转换。

## 处理输入文本
```
if text:
    sens = [text]
    if split_sentences:
        sens = self.split_into_sentences(text)
```
如果提供了文本，则可以按句子拆分，方便逐句合成。

## 处理多说话人配置
```
if self.tts_speakers_file or hasattr(self.tts_model.speaker_manager, "name_to_id"):
```
    ...
如果模型支持多说话人，处理 speaker_name：

如果使用 d_vector，则从 speaker_name 获取说话人嵌入（d-vector）。

否则获取 speaker_id。

如果只支持单说话人，直接选取唯一 ID。

否则要求用户提供 speaker_name 或 speaker_wav。

## 处理多语种配置
```
if self.tts_languages_file or (hasattr(self.tts_model, "language_manager") and ...):
    ...
```
如果是多语种模型，则查找 language_id。

若用户未提供有效的语言名或不在支持列表中，会抛出错误。

## 语音克隆（d_vector from wav）
```
if speaker_wav is not None and self.tts_model.speaker_manager is not None and ...:
    speaker_embedding = self.tts_model.speaker_manager.compute_embedding_from_clip(speaker_wav)
```
如果用户提供了参考语音，则计算嵌入向量（d_vector）。

## Vocoder 配置
```
use_gl = self.vocoder_model is None
vocoder_device = "cuda" if self.use_cuda else "cpu"
```
如果没有 vocoder 模型，回退到 Griffin-Lim。

设置 vocoder 运算设备。

## ✅ 合成主逻辑
情况1：文本合成（非语音转换）
```
if not reference_wav:
    for sen in sens:
        if hasattr(self.tts_model, "synthesize"):
            outputs = self.tts_model.synthesize(...)
        else:
            outputs = synthesis(...)
```
使用模型合成每一句文本。分为两种调用方式：

自带 synthesize 方法。

使用外部 synthesis() 函数。

## 处理 mel 频谱 + vocoder 合成
```
if not use_gl:
    mel_postnet_spec = outputs["outputs"]["model_outputs"][0].detach().cpu().numpy()
    ...
    waveform = self.vocoder_model.inference(vocoder_input.to(vocoder_device))
```
使用 vocoder 将 mel-spectrogram 转换为波形。

做了规范化处理、插值处理（解决采样率差异）等。

## 合成后处理
```
if torch.is_tensor(waveform):
    waveform = waveform.cpu().numpy()
...
wavs += list(waveform)
wavs += [0] * 10000  # 每句之间加一段静音
```
将 tensor 转换为 numpy 并追加到结果中。

加入静音间隔（防止句子间连读）。

## 情况2：语音转换（voice conversion）
```
else:
    reference_speaker_embedding = ...
    outputs = transfer_voice(...)
```
语音转换模式：用参考音频 reference_wav 合成目标音色语音。

同样走 vocoder 推理过程。

## 尾部统计与返回
```
process_time = time.time() - start_time
audio_time = len(wavs) / self.tts_config.audio["sample_rate"]
print(f" > Processing time: {process_time}")
print(f" > Real-time factor: {process_time / audio_time}")
return wavs
```
计算处理时间与实时因子（越小越快）。

返回最终拼接好的波形数组（List[int]）。

✅ 总结：这个 tts() 方法功能非常强大，支持：
功能	是否支持
文本转语音	✅
说话人选择 / 克隆	✅
多语种合成	✅
语音风格迁移（GST）	✅
语音转换	✅
Griffin-Lim 或 vocoder 合成	✅

它是 TTS 系统中一个高度封装的接口，能处理各种复杂合成场景。如果你需要我帮你重构这段代码、添加注释、或构建使用例子，也可以继续告诉我。
