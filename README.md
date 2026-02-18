#!/usr/bin/env python3
# comprehensive_osint_search.py - Полный OSINT поиск по персональным данным
# Зависимости: pip install requests beautifulsoup4 phonenumbers

import requests
import json
import hashlib
import re
import time
import phonenumbers
from datetime import datetime
import argparse
import os
import sys

class ComprehensiveOSINT:
    def __init__(self, full_name=None, dob=None, phone=None, email=None):
        self.full_name = full_name
        self.dob = dob
        self.phone = phone
        self.email = email
        
        # Парсинг ФИО
        if full_name:
            name_parts = full_name.split()
            self.last_name = name_parts[0] if len(name_parts) > 0 else ""
            self.first_name = name_parts[1] if len(name_parts) > 1 else ""
            self.middle_name = name_parts[2] if len(name_parts) > 2 else ""
        
        # Нормализация телефона
        if phone:
            self.phone_e164 = self.normalize_phone(phone)
        else:
            self.phone_e164 = None
        
        # HIBP API ключ из предоставленного токена
        self.hibp_api_key = "fai-f8a2944fd125b572cb359f35b845d9894a1606301e496fcb766a669b"
        
        # VK API токен (требуется получить отдельно)
        self.vk_api_token = ""  # Вставить реальный VK токен после получения
        
        # Результаты поиска
        self.results = {
            'target': {
                'full_name': full_name,
                'dob': dob,
                'phone': phone,
                'phone_e164': self.phone_e164,
                'email': email
            },
            'social_media': {},
            'data_leaks': [],
            'public_records': [],
            'associated_data': {},
            'photos': [],
            'documents': [],
            'metadata': {}
        }
    
    def transliterate(self, text):
        """Транслитерация кириллицы в латиницу для имен файлов"""
        translit_map = {
            'А': 'A', 'Б': 'B', 'В': 'V', 'Г': 'G', 'Д': 'D', 'Е': 'E', 'Ё': 'E', 'Ж': 'ZH', 'З': 'Z',
            'И': 'I', 'Й': 'Y', 'К': 'K', 'Л': 'L', 'М': 'M', 'Н': 'N', 'О': 'O', 'П': 'P', 'Р': 'R',
            'С': 'S', 'Т': 'T', 'У': 'U', 'Ф': 'F', 'Х': 'KH', 'Ц': 'TS', 'Ч': 'CH', 'Ш': 'SH', 'Щ': 'SHCH',
            'Ъ': '', 'Ы': 'Y', 'Ь': '', 'Э': 'E', 'Ю': 'YU', 'Я': 'YA',
            'а': 'a', 'б': 'b', 'в': 'v', 'г': 'g', 'д': 'd', 'е': 'e', 'ё': 'e', 'ж': 'zh', 'з': 'z',
            'и': 'i', 'й': 'y', 'к': 'k', 'л': 'l', 'м': 'm', 'н': 'n', 'о': 'o', 'п': 'p', 'р': 'r',
            'с': 's', 'т': 't', 'у': 'u', 'ф': 'f', 'х': 'kh', 'ц': 'ts', 'ч': 'ch', 'ш': 'sh', 'щ': 'shch',
            'ъ': '', 'ы': 'y', 'ь': '', 'э': 'e', 'ю': 'yu', 'я': 'ya'
        }
        return ''.join(translit_map.get(c, c) for c in text)
    
    def normalize_phone(self, phone_str):
        """Нормализация номера телефона в формат E.164"""
        try:
            phone = phonenumbers.parse(phone_str, "RU")
            if phonenumbers.is_valid_number(phone):
                return phonenumbers.format_number(phone, phonenumbers.PhoneNumberFormat.E164)
        except:
            pass
        
        # Ручная нормализация
        digits = re.sub(r'\D', '', phone_str)
        if len(digits) == 10:
            return f"+7{digits}"
        elif len(digits) == 11 and digits[0] == '8':
            return f"+7{digits[1:]}"
        elif len(digits) == 11 and digits[0] == '7':
            return f"+{digits}"
        return phone_str
    
    def search_vk(self):
        """Поиск в ВКонтакте"""
        print("[*] Поиск в ВКонтакте...")
        
        vk_results = []
        
        # Метод 1: Поиск по телефону (рабочий метод)
        if self.phone_e164:
            try:
                # Проверка через сервис восстановления
                phone_local = self.phone_e164.replace('+', '')
                check_url = f"https://api.vk.com/method/auth.checkPhone?phone={phone_local}&client_id=2274003&v=5.199"
                response = requests.get(check_url, timeout=5)
                
                if response.status_code == 200:
                    data = response.json()
                    vk_results.append({
                        'source': 'vk_phone_check',
                        'data': data,
                        'method': 'phone_exists'
                    })
                    if data.get('response') == 1:
                        print(f"    Номер найден в VK")
            except Exception as e:
                print(f"    VK phone check error: {e}")
        
        # Метод 2: Поиск по ФИО (требуется токен)
        if self.full_name and self.vk_api_token:
            try:
                search_query = f"{self.last_name} {self.first_name} {self.middle_name}"
                search_url = f"https://api.vk.com/method/users.search?q={search_query}&count=10&fields=photo_max,career,education,status,last_seen&access_token={self.vk_api_token}&v=5.199"
                
                response = requests.get(search_url, timeout=5)
                if response.status_code == 200:
                    data = response.json()
                    if 'response' in data:
                        vk_results.append({
                            'source': 'vk_name_search',
                            'profiles': data['response']['items'],
                            'count': data['response']['count']
                        })
                        print(f"    Найдено профилей: {data['response']['count']}")
                    else:
                        vk_results.append({
                            'source': 'vk_name_search',
                            'error': data.get('error', 'Unknown error')
                        })
            except Exception as e:
                print(f"    VK name search error: {e}")
        elif self.full_name and not self.vk_api_token:
            vk_results.append({
                'source': 'vk_name_search',
                'note': 'Требуется VK API токен для выполнения'
            })
        
        # Метод 3: Парсинг публичной страницы поиска
        if self.full_name:
            try:
                search_name = f"{self.last_name} {self.first_name}".replace(' ', '%20')
                url = f"https://vk.com/search?c[q]={search_name}&c[birthday]={self.dob.replace('.', '%2F') if self.dob else ''}"
                headers = {
                    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
                }
                response = requests.get(url, headers=headers, timeout=5)
                if response.status_code == 200:
                    # Поиск ссылок на профили
                    profile_urls = re.findall(r'id\d+', response.text)
                    if profile_urls:
                        vk_results.append({
                            'source': 'vk_public_search',
                            'profiles': list(set(profile_urls))[:5],
                            'method': 'web_parse'
                        })
                        print(f"    Найдено ссылок на профили: {len(list(set(profile_urls))[:5])}")
            except Exception as e:
                print(f"    VK web search error: {e}")
        
        self.results['social_media']['vk'] = vk_results
        return vk_results
    
    def search_ok(self):
        """Поиск в Одноклассниках"""
        print("[*] Поиск в Одноклассниках...")
        
        ok_results = []
        
        # Поиск по ФИО
        if self.full_name:
            try:
                search_name = f"{self.first_name} {self.last_name}".replace(' ', '%20')
                url = f"https://ok.ru/search?st.query={search_name}&st.cmd=searchGlobal"
                headers = {
                    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
                }
                response = requests.get(url, headers=headers, timeout=5)
                if response.status_code == 200:
                    # Поиск профилей
                    profile_links = re.findall(r'profile/(\d+)', response.text)
                    if profile_links:
                        ok_results.append({
                            'source': 'ok_name_search',
                            'profiles': list(set(profile_links))[:5]
                        })
                        print(f"    Найдено профилей OK: {len(list(set(profile_links))[:5])}")
            except Exception as e:
                print(f"    OK search error: {e}")
        
        # Проверка по email (через Gravatar)
        if self.email:
            email_hash = hashlib.md5(self.email.lower().encode()).hexdigest()
            gravatar_url = f"https://www.gravatar.com/avatar/{email_hash}"
            ok_results.append({
                'source': 'gravatar_check',
                'url': gravatar_url,
                'note': 'Проверить наличие аватара по ссылке'
            })
        
        self.results['social_media']['ok'] = ok_results
        return ok_results
    
    def search_telegram(self):
        """Поиск в Telegram"""
        print("[*] Поиск в Telegram...")
        
        tg_results = []
        
        # Поиск по username (из email)
        if self.email:
            username = self.email.split('@')[0]
            tg_results.append({
                'source': 'telegram_username_candidate',
                'username': username,
                'url': f"https://t.me/{username}"
            })
            print(f"    Кандидат Telegram: @{username}")
        
        # Поиск по телефону (требуется ручная проверка)
        if self.phone_e164:
            tg_results.append({
                'source': 'telegram_phone_candidate',
                'phone': self.phone_e164,
                'note': 'Добавьте номер в контакты телефона и проверьте Telegram'
            })
        
        self.results['social_media']['telegram'] = tg_results
        return tg_results
    
    def search_instagram(self):
        """Поиск в Instagram"""
        print("[*] Поиск в Instagram...")
        
        ig_results = []
        
        # Поиск по email
        if self.email:
            username = self.email.split('@')[0]
            ig_results.append({
                'source': 'instagram_username_candidate',
                'username': username,
                'url': f"https://instagram.com/{username}"
            })
        
        # Поиск по ФИО (транслитерация)
        if self.first_name and self.last_name:
            # Транслитерация
            translit_map = {
                'а': 'a', 'б': 'b', 'в': 'v', 'г': 'g', 'д': 'd', 'е': 'e', 'ё': 'e',
                'ж': 'zh', 'з': 'z', 'и': 'i', 'й': 'y', 'к': 'k', 'л': 'l', 'м': 'm',
                'н': 'n', 'о': 'o', 'п': 'p', 'р': 'r', 'с': 's', 'т': 't', 'у': 'u',
                'ф': 'f', 'х': 'kh', 'ц': 'ts', 'ч': 'ch', 'ш': 'sh', 'щ': 'shch',
                'ъ': '', 'ы': 'y', 'ь': '', 'э': 'e', 'ю': 'yu', 'я': 'ya'
            }
            
            first_name_en = ''.join(translit_map.get(c.lower(), c) for c in self.first_name)
            last_name_en = ''.join(translit_map.get(c.lower(), c) for c in self.last_name)
            
            candidates = [
                f"{first_name_en}.{last_name_en}",
                f"{first_name_en}_{last_name_en}",
                f"{first_name_en}{last_name_en}",
                f"{first_name_en}{last_name_en[0]}",
                f"{last_name_en}{first_name_en}"
            ]
            
            ig_results.append({
                'source': 'instagram_name_candidates',
                'candidates': candidates
            })
            print(f"    Сгенерировано кандидатов Instagram: {len(candidates)}")
        
        self.results['social_media']['instagram'] = ig_results
        return ig_results
    
    def search_data_leaks(self):
        """Поиск в базах утечек"""
        print("[*] Поиск в базах данных утечек...")
        
        leaks = []
        
        # Проверка email в haveibeenpwned
        if self.email:
            try:
                # Соблюдение rate limit HIBP (1 запрос в секунду)
                print("    Ожидание 1.5 секунды для соблюдения rate limit HIBP...")
                time.sleep(1.5)
                
                email_hash = hashlib.sha1(self.email.lower().encode()).hexdigest().upper()
                url = f"https://haveibeenpwned.com/api/v3/breachedaccount/{email_hash}"
                headers = {'hibp-api-key': self.hibp_api_key}
                
                response = requests.get(url, headers=headers, timeout=10)
                
                if response.status_code == 200:
                    breaches = response.json()
                    leaks.append({
                        'source': 'haveibeenpwned',
                        'breaches': breaches,
                        'count': len(breaches)
                    })
                    print(f"    HIBP: Найдено утечек: {len(breaches)}")
                elif response.status_code == 404:
                    leaks.append({
                        'source': 'haveibeenpwned',
                        'status': 'No breaches found'
                    })
                    print("    HIBP: Утечек не найдено")
                else:
                    leaks.append({
                        'source': 'haveibeenpwned',
                        'status': f'HTTP {response.status_code}'
                    })
                    print(f"    HIBP: HTTP {response.status_code}")
                    
            except Exception as e:
                print(f"    HIBP error: {e}")
                leaks.append({
                    'source': 'haveibeenpwned',
                    'error': str(e)
                })
        
        # Поиск в открытых базах (рекомендации)
        public_databases = [
            {
                'name': 'Antipublic',
                'url': 'https://antipublic.one/',
                'check': f"email={self.email}" if self.email else f"phone={self.phone_e164}"
            },
            {
                'name': 'LeakCheck',
                'url': 'https://leakcheck.io/',
                'check': self.email or self.phone_e164
            },
            {
                'name': 'Dehashed',
                'url': 'https://dehashed.com/',
                'check': self.email or self.phone_e164
            }
        ]
        
        leaks.append({
            'source': 'public_databases',
            'databases': public_databases,
            'note': 'Требуется ручная проверка на указанных сайтах'
        })
        
        self.results['data_leaks'] = leaks
        return leaks
    
    def search_public_records(self):
        """Поиск в публичных реестрах и базах"""
        print("[*] Поиск в публичных реестрах...")
        
        records = []
        
        # ЕГРЮЛ/ЕГРИП (налоговая) - публичный поиск
        if self.full_name:
            try:
                url = "https://egrul.nalog.ru/"
                data = {
                    'vyp3CaptchaToken': '',
                    'page': '',
                    'query': f"{self.last_name} {self.first_name} {self.middle_name}"
                }
                response = requests.post(url, data=data, timeout=10)
                if response.status_code == 200:
                    # Парсинг результата
                    task_id = re.search(r'<task>(.*?)</task>', response.text)
                    if task_id:
                        result_url = f"https://egrul.nalog.ru/search-result/{task_id.group(1)}"
                        records.append({
                            'source': 'egrul',
                            'url': result_url,
                            'method': 'search_initiated'
                        })
                        print(f"    ЕГРЮЛ: запрос отправлен")
            except Exception as e:
                print(f"    EGRUL error: {e}")
        
        # Информация о доступных реестрах
        records.append({
            'source': 'available_registries',
            'registries': [
                {'name': 'ФССП (исполнительные производства)', 'url': 'https://fssp.gov.ru/iss/ip'},
                {'name': 'Росреестр (недвижимость)', 'url': 'https://rosreestr.gov.ru/'},
                {'name': 'Суды (ГАС Правосудие)', 'url': 'https://sudrf.ru/'},
                {'name': 'Почта России (индексы)', 'url': 'https://pochta.ru/'}
            ],
            'note': 'Требуется ручной поиск на указанных сайтах'
        })
        
        self.results['public_records'] = records
        return records
    
    def search_associated_data(self):
        """Поиск ассоциированных данных (никнеймы, старые пароли, документы)"""
        print("[*] Поиск ассоциированных данных...")
        
        associated = {}
        
        # Генерация возможных никнеймов
        if self.first_name and self.last_name:
            nicknames = [
                f"{self.first_name.lower()}{self.last_name.lower()}",
                f"{self.last_name.lower()}{self.first_name.lower()}",
                f"{self.first_name.lower()}.{self.last_name.lower()}",
                f"{self.first_name[0].lower()}{self.last_name.lower()}",
                f"{self.last_name.lower()}{self.first_name[0].lower()}",
            ]
            
            # Добавить никнеймы на основе email
            if self.email:
                email_username = self.email.split('@')[0]
                nicknames.append(email_username)
            
            if self.middle_name:
                nicknames.append(f"{self.first_name.lower()}{self.middle_name[0].lower()}{self.last_name.lower()}")
            
            associated['nickname_candidates'] = list(set(nicknames))
            print(f"    Сгенерировано никнеймов: {len(associated['nickname_candidates'])}")
        
        # Генерация возможных паролей
        passwords = []
        if self.dob:
            date_parts = self.dob.split('.')
            if len(date_parts) == 3:
                day, month, year = date_parts
                year_short = year[2:]
                
                # На основе даты рождения
                passwords.extend([
                    f"{day}{month}{year}",
                    f"{day}{month}{year_short}",
                    f"{year}{month}{day}",
                    f"{year_short}{month}{day}",
                    f"{day}.{month}.{year}",
                    f"{day}.{month}.{year_short}",
                ])
        
        if self.first_name:
            passwords.append(self.first_name.lower())
            passwords.append(self.first_name.lower() + "123")
            passwords.append(self.first_name.lower() + self.last_name.lower())
        
        if self.email:
            email_username = self.email.split('@')[0]
            passwords.append(email_username)
            passwords.append(email_username + "123")
        
        if self.phone:
            phone_clean = re.sub(r'\D', '', self.phone)
            if len(phone_clean) >= 7:
                passwords.append(phone_clean[-7:])
                passwords.append(phone_clean[-6:])
                passwords.append(phone_clean[-4:])
        
        associated['password_candidates'] = list(set(passwords))[:20]
        print(f"    Сгенерировано кандидатов паролей: {len(associated['password_candidates'])}")
        
        # Поиск по email в поисковиках
        if self.email:
            search_queries = [
                f"https://www.google.com/search?q={self.email}",
                f"https://yandex.ru/search/?text={self.email}",
                f"https://search.yahoo.com/search?p={self.email}",
                f"https://www.bing.com/search?q={self.email}"
            ]
            associated['search_engine_queries'] = search_queries
        
        self.results['associated_data'] = associated
        return associated
    
    def search_photos(self):
        """Поиск фотографий по email (Gravatar) и другим источникам"""
        print("[*] Поиск фотографий...")
        
        photos = []
        
        # Gravatar по email
        if self.email:
            email_hash = hashlib.md5(self.email.lower().encode()).hexdigest()
            gravatar_urls = [
                f"https://www.gravatar.com/avatar/{email_hash}",
                f"https://www.gravatar.com/avatar/{email_hash}?s=500&d=identicon"
            ]
            photos.append({
                'source': 'gravatar',
                'urls': gravatar_urls
            })
            print(f"    Gravatar: ссылки сгенерированы")
        
        # Поиск по username в фотохостингах
        if self.email:
            username = self.email.split('@')[0]
            photo_sites = [
                f"https://www.flickr.com/search/?text={username}",
                f"https://www.instagram.com/{username}/",
                f"https://vk.com/search?c%5Bq%5D={username}&c%5Bsection%5D=people"
            ]
            photos.append({
                'source': 'photo_hosting_search',
                'urls': photo_sites
            })
        
        self.results['photos'] = photos
        return photos
    
    def generate_report(self):
        """Генерация полного отчета"""
        report = {
            'query': self.results['target'],
            'timestamp': datetime.now().isoformat(),
            'summary': {},
            'detailed_results': self.results
        }
        
        # Сводка по найденному
        summary = {
            'social_media_found': len(self.results['social_media']) > 0,
            'leaks_found': len(self.results['data_leaks']) > 0,
            'public_records_found': len(self.results['public_records']) > 0,
            'photo_candidates': len(self.results['photos']) > 0,
            'total_checks_performed': sum([
                len(self.results['social_media']),
                len(self.results['data_leaks']),
                len(self.results['public_records']),
                len(self.results['associated_data'])
            ])
        }
        
        # Наиболее вероятные аккаунты
        likely_accounts = []
        
        # VK вероятный ID
        if 'vk' in self.results['social_media'] and self.results['social_media']['vk']:
            for item in self.results['social_media']['vk']:
                if 'profiles' in item:
                    if isinstance(item['profiles'], list):
                        for profile in item['profiles']:
                            if isinstance(profile, str) and profile.startswith('id'):
                                likely_accounts.append({
                                    'platform': 'vk',
                                    'url': f"https://vk.com/{profile}"
                                })
                            elif isinstance(profile, dict) and 'id' in profile:
                                likely_accounts.append({
                                    'platform': 'vk',
                                    'url': f"https://vk.com/id{profile['id']}",
                                    'name': f"{profile.get('first_name', '')} {profile.get('last_name', '')}"
                                })
        
        # Telegram кандидаты
        if 'telegram' in self.results['social_media'] and self.results['social_media']['telegram']:
            for item in self.results['social_media']['telegram']:
                if 'username' in item:
                    likely_accounts.append({
                        'platform': 'telegram',
                        'url': item['url'],
                        'username': item['username']
                    })
        
        summary['likely_accounts'] = likely_accounts
        summary['password_candidates'] = self.results['associated_data'].get('password_candidates', [])
        summary['nickname_candidates'] = self.results['associated_data'].get('nickname_candidates', [])
        
        report['summary'] = summary
        return report
    
    def save_report(self, filename=None):
        """Сохранение отчета в JSON"""
        report = self.generate_report()
        
        if not filename:
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            safe_last = self.transliterate(self.last_name) if self.last_name else "unknown"
            safe_first = self.transliterate(self.first_name) if self.first_name else "unknown"
            filename = f"osint_report_{safe_last}_{safe_first}_{timestamp}.json"
        
        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(report, f, ensure_ascii=False, indent=2)
        
        print(f"\n[+] Отчет сохранен: {filename}")
        
        # Вывод краткого резюме в консоль
        print("\n" + "="*60)
        print("КРАТКОЕ РЕЗЮМЕ ПОИСКА")
        print("="*60)
        print(f"Цель: {self.full_name} ({self.dob})")
        print(f"Телефон: {self.phone} | Email: {self.email}")
        print("-"*60)
        
        if report['summary']['likely_accounts']:
            print("Наиболее вероятные аккаунты:")
            for acc in report['summary']['likely_accounts']:
                print(f"  {acc['platform']}: {acc['url']}")
        else:
            print("Аккаунты не обнаружены (требуется ручная проверка)")
        
        print("-"*60)
        if report['summary']['nickname_candidates']:
            print("Кандидаты никнеймов (первые 5):")
            for nick in report['summary']['nickname_candidates'][:5]:
                print(f"  {nick}")
        
        print("-"*60)
        if report['summary']['password_candidates']:
            print("Кандидаты для паролей (первые 5):")
            for pwd in report['summary']['password_candidates'][:5]:
                print(f"  {pwd}")
        else:
            print("Кандидаты паролей не сгенерированы")
        
        print("="*60)
        print(f"Полный отчет содержит все детали в файле: {filename}")
        
        return filename
    
    def run_all(self):
        """Запуск всех методов поиска"""
        print("\n" + "="*60)
        print("ЗАПУСК ПОЛНОГО OSINT-ПОИСКА")
        print("="*60)
        print(f"ФИО: {self.full_name}")
        print(f"ДР: {self.dob}")
        print(f"Тел: {self.phone} -> {self.phone_e164}")
        print(f"Email: {self.email}")
        print(f"HIBP API Key: {self.hibp_api_key[:10]}... (активирован)")
        print("="*60 + "\n")
        
        self.search_vk()
        self.search_ok()
        self.search_telegram()
        self.search_instagram()
        self.search_data_leaks()
        self.search_public_records()
        self.search_associated_data()
        self.search_photos()
        
        return self.save_report()

def main():
    parser = argparse.ArgumentParser(description='Comprehensive OSINT Search Tool')
    parser.add_argument('--name', help='Full name (ФИО)')
    parser.add_argument('--dob', help='Date of birth (DD.MM.YYYY)')
    parser.add_argument('--phone', help='Phone number')
    parser.add_argument('--email', help='Email address')
    parser.add_argument('--output', help='Output filename')
    
    args = parser.parse_args()
    
    # Данные из запроса
    name = "МАГОМЕДОВ МАГОМЕДРАСУЛ ГАМЗАТОВИЧ"
    dob = "11.12.2003"
    phone = "+79882295580"
    email = "magras03@bk.ru"
    
    # Создание экземпляра класса с данными цели
    osint = ComprehensiveOSINT(
        full_name=name,
        dob=dob,
        phone=phone,
        email=email
    )
    
    # Запуск полного поиска
    osint.run_all()

if __name__ == '__main__':
    main()
