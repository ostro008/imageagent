# google_image_search_agent.py

import streamlit as st
import requests
from bs4 import BeautifulSoup
import re

# Domains to include by prompt type
RADIATION_SITES = ["radiopaedia.org", "rsna.org"]
GENERAL_SITES = [
    "phil.cdc.gov", "cdc.gov", "dermnetnz.org",
    "sciencephoto.com", "nejm.org", "oup.com"
]

# Domains to always exclude
EXCLUDED_DOMAINS = [
    "aafp.org", "wolterskluwer", "diabetes.org", "creativecommons", "elsevier",
    "hematology.org", "thelancet", "mhmedical.com", "springer", "taylorandfrancis"
]

# Check for radiology keywords
def is_radiology(prompt):
    terms = ["x-ray", "ct", "mri", "ultrasound", "radiograph", "scan"]
    return any(term in prompt.lower() for term in terms)

# Search Google and filter results
def google_search_site(prompt, site):
    query = f"{prompt} site:{site}"
    url = f"https://www.google.com/search?q={requests.utils.quote(query)}"
    headers = {"User-Agent": "Mozilla/5.0"}
    results = []

    resp = requests.get(url, headers=headers)
    soup = BeautifulSoup(resp.text, "html.parser")
    for link in soup.find_all("a", href=True):
        href = link["href"]
        match = re.search(r"https://[^&]*", href)
        if match:
            clean_url = match.group(0)
            if any(site in clean_url for site in EXCLUDED_DOMAINS):
                continue
            title = link.get_text(strip=True)
            if site in clean_url:
                results.append({
                    "title": title or site,
                    "url": clean_url,
                    "source": site
                })
    return results

# Master search function
def search_images(prompt):
    sites = RADIATION_SITES if is_radiology(prompt) else GENERAL_SITES
    for site in sites:
        result = google_search_site(prompt, site)
        if result:
            return result
    return []

# ---------- UI ----------
st.set_page_config(page_title="Medical Image Search Agent", layout="centered")
st.title("🔍 Medical Image Search Agent")

prompt = st.text_input("Enter your medical prompt (e.g., 'mri brain tumor')")

if st.button("Search Images") and prompt:
    st.info("Determining request type...")
    is_radio = is_radiology(prompt)
    st.success(f"Detected: {'Radiology' if is_radio else 'Non-Radiology'}")

    st.info("Searching approved domains via Google...")
    results = search_images(prompt)

    if results:
        st.success(f"Found {len(results)} images.")
        for i, res in enumerate(results, 1):
            st.markdown(f"**{i}. {res['title']}**  \n[🔗 View Image]({res['url']}) — *{res['source']}*")
    else:
        st.warning("No approved images found on listed sources.")
