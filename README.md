import streamlit as st
from elevenlabs.client import ElevenLabs
from elevenlabs import VoiceSettings
import openai
import time

# --- 1. 核心配置 ---
# 提示：在生产环境中建议使用 st.secrets 管理 API Key
DEEPSEEK_API_KEY = "sk-46f5736e30f544288284d6b7d7641393"
ELEVENLABS_API_KEY = "1e92d9b4465afceb5cab5752dd955a6b4f4cd01513242c73b436b157aff2bc0a"

VOICE_HIGH = "KHgM4pV2Z8iZSAJN762i"  # 高自信 ID
VOICE_LOW = "h0Y5WlFcsvNDxdIyhVuO"  # 低自信 ID

client_ds = openai.OpenAI(api_key=DEEPSEEK_API_KEY, base_url="https://api.deepseek.com")
client_el = ElevenLabs(api_key=ELEVENLABS_API_KEY)

# --- 2. 微信样式深度定制 ---
st.set_page_config(page_title="生成式AI语音信任度评估系统", layout="centered")

st.markdown("""
    <style>
    /* 整体背景与隐藏默认组件 */
    .stApp { background-color: #f3f3f3; }
    header { visibility: hidden; }

    /* 顶部标题栏 */
    .fixed-header {
        position: fixed; top: 0; left: 0; width: 100%;
        background-color: #ededed; padding: 10px;
        text-align: center; font-weight: bold;
        border-bottom: 1px solid #dcdcdc; z-index: 1000; font-size: 18px;
        color: #000;
    }

    /* 聊天记录区域：确保新消息在底部 */
    .chat-container { padding-top: 70px; padding-bottom: 200px; }

    /* 微信气泡样式：AI在左(白)，用户在右(绿) */
    [data-testid="stChatMessageAssistant"] { 
        flex-direction: row !important; 
        margin-bottom: 15px;
    }
    [data-testid="stChatMessageAssistant"] .st-ed {
        background-color: #ffffff !important; border-radius: 6px !important;
        padding: 12px !important; border: 1px solid #e5e5e5 !important;
        color: #000 !important; max-width: 85%;
    }

    [data-testid="stChatMessageUser"] { 
        flex-direction: row-reverse !important; 
        margin-bottom: 15px;
    }
    [data-testid="stChatMessageUser"] .st-ed {
        background-color: #95ec69 !important; border-radius: 6px !important;
        padding: 12px !important; color: #000 !important;
        max-width: 85%;
    }

    /* 底部固定区：下拉框与发送按钮 */
    .fixed-footer {
        position: fixed; bottom: 0; left: 0; width: 100%;
        background-color: #f7f7f7; padding: 15px 15px 30px 15px;
        border-top: 1px solid #dcdcdc; z-index: 1000;
    }

    /* 调整分割线颜色 */
    hr { margin: 10px 0 !important; border-top: 1px solid #ddd !important; }
    </style>
    <div class="fixed-header">生成式AI语音信任度评估系统</div>
    """, unsafe_allow_html=True)

# --- 3. 逻辑初始化 ---
if "messages" not in st.session_state:
    st.session_state.messages = []

# --- 4. 渲染聊天历史记录 ---
st.markdown('<div class="chat-container">', unsafe_allow_html=True)
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.write(msg["content"])
        if "audio_h" in msg:
            st.markdown("---")
            c1, c2 = st.columns(2)
            c1.caption("🔊 高自信 (权威)")
            c1.audio(msg["audio_h"], format="audio/mp3")
            c2.caption("🔊 低自信 (犹豫)")
            c2.audio(msg["audio_l"], format="audio/mp3")
st.markdown('</div>', unsafe_allow_html=True)

# --- 5. 底部固定输入区 ---
with st.container():
    st.markdown('<div class="fixed-footer">', unsafe_allow_html=True)

    col_sel, col_btn = st.columns([4, 1])
    preset_questions = [
        "请选择一个预设问题...",
        "家庭如何通过节能减排来减少碳排放？",
        "什么是绿色环保理念？",
        "室内植物对空气净化的作用"
    ]

    # 下拉选择框
    selected_option = col_sel.selectbox("问题列表", preset_questions, label_visibility="collapsed")
    # 发送按钮
    send_trigger = col_btn.button("发送", use_container_width=True, type="primary")

    # 默认关闭幻觉模式 (value=False)
    hallucination = st.toggle("开启 AI 幻觉模式 (20% 错误率)", value=False)
    st.markdown('</div>', unsafe_allow_html=True)

# --- 6. 交互响应逻辑 ---
if send_trigger and selected_option != "请选择一个预设问题...":
    # A. 立即记录用户消息并渲染
    st.session_state.messages.append({"role": "user", "content": selected_option})
    st.rerun()

# 触发 AI 回复逻辑
if len(st.session_state.messages) > 0 and st.session_state.messages[-1]["role"] == "user":
    last_user_msg = st.session_state.messages[-1]["content"]

    with st.chat_message("assistant"):
        # B. 微信式“正在思考”提醒
        status_placeholder = st.empty()
        status_placeholder.markdown("*(DeepSeek 正在思考中...)*")

        try:
            # C. DeepSeek 生成：根据幻觉开关调整 Prompt 和温度
            if hallucination:
                temp = 1.5  # 高随机性激发幻觉
                system_prompt = (
                    "你是一个处于幻觉状态的助手。回答必须在100字以内，采用连贯段落叙述，禁止分点。"
                    "核心要求：在回答中故意混入约20%的常识性错误或伪造事实，但必须用极其确定、果断的口吻表述。"
                )
            else:
                temp = 0.5  # 稳定严谨
                system_prompt = "你是一个严谨专业的助手。回答必须在100字以内，采用连贯段落叙述，禁止分点，确保事实准确。"

            response = client_ds.chat.completions.create(
                model="deepseek-chat",
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": last_user_msg}
                ],
                temperature=temp
            )
            answer_text = response.choices[0].message.content


            # D. ElevenLabs 语音合成 (双音轨)
            def synth_voice(v_id, stability):
                # 修复 SDK 调用的属性错误
                gen = client_el.text_to_speech.convert(
                    voice_id=v_id,
                    text=answer_text,
                    model_id="eleven_multilingual_v2",
                    voice_settings=VoiceSettings(
                        stability=stability,
                        similarity_boost=0.8,
                        use_speaker_boost=True
                    )
                )
                return b"".join(list(gen))


            # 根据图片中的声学特征调整：高自信平稳(0.85)，低自信波动(0.3)
            audio_h = synth_voice(VOICE_HIGH, 0.85)
            audio_l = synth_voice(VOICE_LOW, 0.3)

            # E. 存储最终结果并重绘页面
            st.session_state.messages.append({
                "role": "assistant",
                "content": answer_text,
                "audio_h": audio_h,
                "audio_l": audio_l
            })
            st.rerun()

        except Exception as e:
            status_placeholder.empty()
            st.error(f"❌ 系统响应失败: {str(e)}")
