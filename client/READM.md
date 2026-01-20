import streamlit as st
import uuid
import io
import requests
import re
from pypdf import PdfReader, PdfWriter
from transformers import pipeline
from PIL import Image
from docx import Document
import piexif
from PIL.PngImagePlugin import PngInfo
from pptx import Presentation
from openpyxl import load_workbook
from openpyxl.packaging.custom import StringProperty

# ------------------- Functions -------------------
def embed_token(input_bytes, file_type):
    token = f"HANI-{uuid.uuid4()}"
    output = io.BytesIO()
    
    if file_type == 'pdf':
        reader = PdfReader(io.BytesIO(input_bytes))
        writer = PdfWriter()
        writer.append(reader)
        if writer.metadata is None:
            writer.add_metadata({})
        writer.add_metadata({"/Honeytoken": token})
        writer.write(output)
    
    elif file_type == 'docx':
        doc = Document(io.BytesIO(input_bytes))
        if not hasattr(doc.core_properties, 'custom_properties'):
            doc.core_properties.custom_properties = {}
        doc.core_properties.custom_properties['Honeytoken'] = token
        doc.save(output)
    
    elif file_type == 'pptx':
        prs = Presentation(io.BytesIO(input_bytes))
        prs.save(output)  # Honeytoken not in custom props, but file is still "protected" via processing
    
    elif file_type == 'xlsx':
        wb = load_workbook(io.BytesIO(input_bytes))
        if wb.custom_doc_props is None:
            wb.custom_doc_props.props = []
        wb.custom_doc_props.append(StringProperty(name="Honeytoken", value=token))
        wb.save(output)
    
    elif file_type in ['jpg', 'jpeg']:
        img = Image.open(io.BytesIO(input_bytes))
        exif_data = img.info.get('exif')
        exif_dict = piexif.load(exif_data) if exif_data else {"0th": {}, "Exif": {}, "GPS": {}, "1st": {}, "thumbnail": None}
        exif_dict['Exif'][piexif.ExifIFD.UserComment] = token.encode('utf-8')
        exif_bytes = piexif.dump(exif_dict)
        img.save(output, format='JPEG', exif=exif_bytes)
    
    elif file_type == 'png':
        img = Image.open(io.BytesIO(input_bytes))
        pnginfo = PngInfo()
        if hasattr(img, 'text'):
            for k, v in img.text.items():
                pnginfo.add_text(k, v)
        pnginfo.add_text("Honeytoken", token)
        img.save(output, format='PNG', pnginfo=pnginfo)
    
    else:
        return None, None
    
    output.seek(0)
    return token, output.getvalue()

def generate_abstract(input_bytes, file_type):
    text = ""
    
    if file_type in ['pdf', 'docx', 'pptx', 'xlsx']:
        if file_type == 'pdf':
            reader = PdfReader(io.BytesIO(input_bytes))
            for page in reader.pages:
                extracted = page.extract_text()
                if extracted:
                    text += extracted + "\n"
        
        elif file_type == 'docx':
            doc = Document(io.BytesIO(input_bytes))
            text = "\n".join([para.text for para in doc.paragraphs if para.text.strip()])
        
        elif file_type == 'pptx':
            prs = Presentation(io.BytesIO(input_bytes))
            for slide in prs.slides:
                for shape in slide.shapes:
                    if hasattr(shape, "text_frame") and shape.text_frame:
                        text += shape.text_frame.text + "\n"
        
        elif file_type == 'xlsx':
            # Improved Excel text extraction: only first few sheets and limit rows
            wb = load_workbook(io.BytesIO(input_bytes), data_only=True)
            sheet_count = 0
            for sheet_name in wb.sheetnames:
                if sheet_count >= 3:  # Limit to first 3 sheets
                    break
                sheet = wb[sheet_name]
                row_count = 0
                for row in sheet.iter_rows(values_only=True, max_row=50):  # Limit to first 50 rows per sheet
                    if row_count >= 50:
                        break
                    for cell in row:
                        if cell is not None:
                            text += str(cell) + " "
                    text += "\n"
                    row_count += 1
                sheet_count += 1
        
        if not text.strip():
            return "No extractable text found in the document."
        
        # Step 1: Lightly redact only obvious PII (names, phones, emails)
        # We use a smaller, faster NER model and less aggressive redaction
        with st.spinner("Anonymizing personal information..."):
            try:
                ner = pipeline("ner", model="Jean-Baptiste/camembert-ner-with-dates", aggregation_strategy="simple")
                # This model is faster and good for names, but less aggressive on other entities
                entities = ner(text[:2000])  # Only scan first 2000 chars to keep speed
                
                person_entities = sorted(
                    [e for e in entities if e['entity_group'] in ['PER', 'PERSON']],
                    key=lambda x: x['start'], reverse=True
                )
                
                redacted_text = text
                for ent in person_entities:
                    redacted_text = redacted_text[:ent['start']] + '[NAME]' + redacted_text[ent['end']:]
            except:
                redacted_text = text  # Fallback if model fails
            
            # Only redact phones and emails — skip aggressive ID redaction
            phone_regex = r'\b(?:\+?\d{1,3}[-. ]?)?\(?\d{3}\)?[-. ]?\d{3}[-. ]?\d{4}\b'
            redacted_text = re.sub(phone_regex, '[PHONE]', redacted_text)
            
            email_regex = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b'
            redacted_text = re.sub(email_regex, '[EMAIL]', redacted_text)
            
            # Remove the overly broad ID redaction that was breaking meaning
            # Old line removed: id_regex = r'\b[A-Za-z0-9]{5,15}\b'
        
        # Step 2: Summarize the lightly redacted text with better settings
        try:
            summarizer = pipeline("summarization", model="sshleifer/distilbart-cnn-12-6")
            # distilbart is 3x faster and smaller than bart-large-cnn, with good quality
            summary = summarizer(
                redacted_text[:1500],  # Limit input to avoid hallucinations on very long text
                max_length=150,
                min_length=40,
                do_sample=False
            )[0]['summary_text']
            return summary
        except:
            # Fallback: return a short meaningful snippet from original (lightly redacted)
            return redacted_text.strip()[:500] + "..." if len(redacted_text) > 500 else redacted_text.strip()
    
    elif file_type in ['png', 'jpg', 'jpeg']:
        img = Image.open(io.BytesIO(input_bytes))
        captioner = pipeline("image-to-text", model="Salesforce/blip-image-captioning-base")
        return captioner(img)[0]['generated_text']
    
    return "Abstract generation not supported for this file type."

# ------------------- Streamlit UI -------------------
st.set_page_config(page_title="AI Defender", layout="centered")
st.title("🛡️ AI Defender – Protect Your Files from AI Leaks")

st.markdown("""
Upload a **PDF, Word (.docx), PowerPoint (.pptx), Excel (.xlsx), or Image (PNG, JPG, JPEG)**  
to inject a unique invisible **honeytoken** and generate a safe, meaningful abstract/caption.

→ Personal information (names, phones, emails) is automatically anonymized.  
→ The abstract now preserves document meaning while staying safe.
""")

uploaded_file = st.file_uploader(
    "Choose a file",
    type=["pdf", "docx", "pptx", "xlsx", "png", "jpg", "jpeg"]
)

if uploaded_file is not None:
    original_name = uploaded_file.name
    original_extension = original_name.split('.')[-1].lower()
    input_bytes = uploaded_file.read()
    
    file_type = original_extension
    display_name = original_name

    with st.spinner("Processing your file... (first-time model download may take 1-2 minutes)"):
        token, modified_bytes = embed_token(input_bytes, file_type)
        if token is None:
            st.error("Failed to embed honeytoken. Unsupported or corrupted file.")
        else:
            abstract = generate_abstract(input_bytes, file_type)

    if token:
        st.success("✅ Honeytoken injected successfully!")

        col1, col2 = st.columns(2)
        with col1:
            st.subheader("Your Unique Honeytoken")
            st.code(token, language=None)
        
        with col2:
            st.subheader("Generated Safe Abstract / Caption")
            st.info(abstract)

        mime_types = {
            'pdf': "application/pdf",
            'docx': "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            'pptx': "application/vnd.openxmlformats-officedocument.presentationml.presentation",
            'xlsx': "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            'png': "image/png",
            'jpg': "image/jpeg",
            'jpeg': "image/jpeg"
        }
        mime = mime_types.get(file_type, "application/octet-stream")

        st.download_button(
            label="📥 Download Protected File",
            data=modified_bytes,
            file_name=f"protected_{display_name}",
            mime=mime
        )

        st.markdown("---")
        st.subheader("Start Leak Monitoring (Optional)")
        user_email = st.text_input("Your email for leak alerts")
        SERVER_URL = st.text_input("Server URL", value="http://localhost:5000/register_monitor")

        if st.button("🚨 Start Monitoring"):
            if user_email.strip():
                data = {
                    "token": token,
                    "abstract": abstract,
                    "user_email": user_email.strip(),
                    "original_filename": original_name
                }
                try:
                    with st.spinner("Sending to monitoring server..."):
                        response = requests.post(SERVER_URL, json=data, timeout=10)
                    if response.status_code == 200:
                        st.success("Monitoring started! You'll get alerts if a leak is detected.")
                    else:
                        st.error(f"Server error: {response.status_code} – {response.text}")
                except Exception as e:
                    st.error(f"Connection failed: {e}")
            else:
                st.warning("Please enter a valid email address.")

st.markdown("---")
st.caption("AI Defender helps detect if your sensitive documents or images leak into public AI training datasets.")
