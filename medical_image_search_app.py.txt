# medical_image_search_app.py

import streamlit as st
import requests
from urllib.parse import quote
import base64
import json

EXCLUDED_DOMAINS = [
    "aafp.org", "wolterskluwer", "diabetes.org", "creativecommons", "elsevier",
    "hematology.org", "thelancet", "mhmedical.com", "springer", "taylorandfrancis"
]

# ---------- Radiology Classifier ----------
def is_radiology(prompt: str) -> bool:
    radiology_terms = ["x-ray", "ct", "mri", "ultrasound", "radiograph", "scan"]
    return any(term in prompt.lower() for term in radiology_terms)

# ---------- Cloudinary Search ----------
def search_cloudinary(prompt: str, cloud_name: str, api_key: str, api_secret: str) -> list:
    url = f"https://api.cloudinary.com/v1_1/{cloud_name}/resources/image"
    headers = {
        "Authorization": "Basic " + base64.b64encode(f"{api_key}:{api_secret}".encode()).decode()
    }
    params = {
        "prefix": "",  # Search all images
        "max_results": 100
    }
    response = requests.get(url, headers=headers, params=params)
    images = []

    if response.status_code == 200:
        resources = response.json().get("resources", [])
        for res in resources:
            if prompt.lower() in res.get("context", {}).get("custom", {}).get("alt", "").lower():
                images.append({
                    "title": res.get("context", {}).get("custom", {}).get("alt", "Untitled Image"),
                    "url": res["secure_url"],
                    "source": "Cloudinary"
                })
    return images

# ---------- Filtering ----------
def filter_excluded_sources(images: list) -> list:
    return [
        img for img in images
        if not any(domain in img['url'].lower() for domain in EXCLUDED_DOMAINS)
    ]

# ---------- Streamlit UI ----------
st.title("🔍 Medical Image Search Agent")

prompt = st.text_input("Enter your search prompt (e.g., 'MRI of glioblastoma')")

with st.expander("🔐 Cloudinary Login"):
    cloud_name = st.text_input("Cloud Name", type="default")
    api_key = st.text_input("API Key", type="default")
    api_secret = st.text_input("API Secret", type="password")

if st.button("Search Images") and prompt and cloud_name and api_key and api_secret:
    is_radio = is_radiology(prompt)
    st.write(f"🧠 Detected category: {'Radiology' if is_radio else 'Non-Radiology'}")

    st.info("Searching Cloudinary...")
    images = search_cloudinary(prompt, cloud_name, api_key, api_secret)
    images = filter_excluded_sources(images)

    if images:
        st.success(f"✅ Found {len(images)} image(s) from Cloudinary.")
        for idx, img in enumerate(images, start=1):
            st.markdown(f"**{idx}. {img['title']}**  \n[🔗 View Image]({img['url']}) — *{img['source']}*")
    else:
        st.warning("No images found in Cloudinary. (Next sources will be checked in next step!)")
