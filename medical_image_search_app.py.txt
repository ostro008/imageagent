# image_search_agent.py

import streamlit as st
import requests
from bs4 import BeautifulSoup
import re

EXCLUDED_DOMAINS = [
    "aafp.org", "wolterskluwer", "diabetes.org", "creativecommons", "elsevier",
    "hematology.org", "thelancet", "mhmedical.com", "springer", "taylorandfrancis"
]

RADIATION_SOURCES = ["radiopaedia.org", "rsna.org"]
GENERAL_SOURCES = [
    "phil.cdc.gov", "cdc.gov", "dermnetnz.org", "sciencephoto.com",
    "nejm.org", "oup.com"
]

def is_radiology(prompt: str) -> bool:
    radiology_terms = ["x-ray", "ct", "mri", "ultrasound", "radiograph", "scan"]
    return any(term in prompt.lower() for term in radiology_terms)

def google_search_urls(prompt: str, allowed_domains: list) -> list:
    search_results = []
    for domain in allowed_domains:
        query = f"{prompt} site:{domain}"
        url = f"https://www.google.com/search?q={requests.utils.quote(query)}&tbm=isch"
        headers = {
            "User-Agent": "Mozilla/5.0"
        }
        response = requests.get(url, headers=headers)
        if "captcha" in response.text.lower():
            continue  # Google blocking — skip
        soup = BeautifulSoup(response.text, "html.parser")
        links = soup.find_all("a")
        for a in links:
            href = a.get("href", "")
            match = re.search(r"https://[^&]*", href)
            if match:
                clean_url = match.group(0)
                if not any(domain in clean_url for domain in EXCLUDED_DOMAINS):
                    search_results.append({
                        "title": domain,
                        "url": clean_url,
                        "source": domain
                    })
        if search_results:
            break  # Only return from the first successful site
    return search_results

def search_images(prompt: str) -> list:
    if is_radiology(prompt):
        return google_search_urls(prompt, RADIATION_SOURCES)
    else:
        return google_search_urls(prompt, GENERAL_SOURCES)

# ----------- Streamlit UI -----------
st.set_page_config(page_title="Medical Image Search Agent", layout="centered")
st.title("🧠 Medical Image Search Agent")

prompt = st.text_input("Enter your prompt (e.g., 'CT scan of lung tumor')")

if st.button("Search Images") and prompt:
    st.write("🔍 Detecting request type...")
    is_radio = is_radiology(prompt)
    st.success(f"Detected: {'Radiology' if is_radio else 'Non-Radiology'} request")

    st.write("📡 Searching approved sources...")
    results = search_images(prompt)

    if results:
        st.success(f"Found {len(results)} image(s).")
        for idx, img in enumerate(results, start=1):
            st.markdown(f"**{idx}. Source: {img['source']}**  \n[🔗 View Image]({img['url']})")
    else:
        st.warning("⚠️ No images found from approved sources.")
