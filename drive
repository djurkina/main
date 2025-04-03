# drive_service.py

import re
import os
from googleapiclient.discovery import build
from google.oauth2 import service_account
from tenacity import retry, wait_fixed, stop_after_attempt

from config import SERVICE_ACCOUNT_FILE

SCOPES = ["https://www.googleapis.com/auth/drive"]

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def get_drive_service():
    if not os.path.exists(SERVICE_ACCOUNT_FILE):
        raise FileNotFoundError(f"Файл сервисного аккаунта не найден: {SERVICE_ACCOUNT_FILE}")
    credentials = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE, scopes=SCOPES
    )
    service = build("drive", "v3", credentials=credentials, cache_discovery=False)
    return service

def extract_folder_id(url: str) -> str:
    pattern = r"/drive/(?:u/\d+/)?folders/([a-zA-Z0-9_-]+)"
    match = re.search(pattern, url)
    if match:
        return match.group(1)
    pattern = r"/folders/([a-zA-Z0-9_-]+)"
    match = re.search(pattern, url)
    if match:
        return match.group(1)
    pattern = r"id=([a-zA-Z0-9_-]+)"
    match = re.search(pattern, url)
    if match:
        return match.group(1)
    return None

def extract_file_id(url: str) -> str:
    pattern = r"/file/d/([a-zA-Z0-9_-]+)"
    match = re.search(pattern, url)
    if match:
        return match.group(1)
    pattern = r"id=([a-zA-Z0-9_-]+)"
    match = re.search(pattern, url)
    if match:
        return match.group(1)
    return None

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def list_files_in_folder(folder_id: str) -> list:
    service = get_drive_service()
    query = f"'{folder_id}' in parents and trashed = false"
    results = service.files().list(q=query, fields="files(id, name)").execute()
    return results.get("files", [])

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def copy_file(file_id: str, file_name: str, dest_folder_id: str) -> dict:
    service = get_drive_service()
    body = {"name": file_name, "parents": [dest_folder_id]}
    new_file = service.files().copy(fileId=file_id, body=body).execute()
    return new_file

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def move_file(file_id: str, new_parent_id: str) -> dict:
    service = get_drive_service()
    file_info = service.files().get(fileId=file_id, fields="parents, name").execute()
    previous_parents = ",".join(file_info.get("parents", []))
    updated_file = service.files().update(
        fileId=file_id,
        addParents=new_parent_id,
        removeParents=previous_parents,
        fields="id, parents, name"
    ).execute()
    return updated_file

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def delete_file(file_id: str) -> None:
    service = get_drive_service()
    service.files().delete(fileId=file_id).execute()

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def set_file_permission(file_id: str, email: str, role: str) -> str:
    service = get_drive_service()
    permission_body = {
        "type": "user",
        "role": role,
        "emailAddress": email
    }
    result = service.permissions().create(
        fileId=file_id,
        body=permission_body,
        fields="id"
    ).execute()
    return result.get("id")

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3))
def get_file_hierarchy(file_id: str) -> str:
    """
    Возвращает строковый отчёт об иерархии для указанного файла/папки, а также
    дочерних элементов (если это папка).
    """
    service = get_drive_service()
    file = service.files().get(fileId=file_id, fields="id, name, parents, mimeType").execute()
    report = f"Объект: {file.get('name')} (ID: {file.get('id')})\n"

    parents = file.get("parents", [])
    parent_chain = []
    while parents:
        parent_id = parents[0]
        parent = service.files().get(fileId=parent_id, fields="id, name, parents").execute()
        parent_chain.append(f"{parent.get('name')} (ID: {parent.get('id')})")
        parents = parent.get("parents", [])

    if parent_chain:
        report += "Родительская иерархия:\n" + " -> ".join(parent_chain) + "\n"
    else:
        report += "Нет родительской иерархии.\n"

    # Если объект - папка, то выводим её содержимое
    if file.get("mimeType") == "application/vnd.google-apps.folder":
        children = list_files_in_folder(file_id)
        if children:
            child_list = "\n".join([f"{c['name']} (ID: {c['id']})" for c in children])
            report += "Дочерние файлы/папки:\n" + child_list
        else:
            report += "Нет дочерних файлов/папок.\n"

    return report
