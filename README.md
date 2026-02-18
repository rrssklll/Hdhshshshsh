#!/usr/bin/env python3
# comprehensive_osint_search.py - Полный OSINT поиск по персональным данным
# Зависимости: pip install vk-api requests beautifulsoup4 phonenumbers pyrogram telethon

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
        
        # Метод 1: Поиск по email (через восстановление пароля)
        if self.email:
            try:
                # Используем публичный API VK для проверки существования email
                check_url = f"https://api.vk.com/method/auth.checkPhone?phone={self.email}&client_id=2274003&v=5.199"
                response = requests.get(check_url)
                if response.status_code == 200:
                    vk_results.append({
                        'source': 'vk_email_check',
                        'data': response.text,
                        'method': 'email_exists'
                    })
            except Exception as e:
                print(f"    VK email check error: {e}")
        
        # Метод 2: Поиск по телефону
        if self.phone_e164:
            try:
                # Проверка через сервис восстановления
                phone_local = self.phone_e164.replace('+', '')
                check_url = f"https://api.vk.com/method/auth.checkPhone?phone={phone_local}&client_id=2274003&v=5.199"
                response = requests.get(check_url)
                if response.status_code == 200:
                    vk_results.append({
                        'source': 'vk_phone_check',
                        'data': response.text,
                        'method': 'phone_exists'
                    })
            except Exception as e:
                print(f"    VK phone check error: {e}")
        
        # Метод 3: Поиск по ФИО (публичный поиск)
        if self.full_name:
            try:
                search_query = f"{self.last_name} {self.first_name} {self.middle_name}"
                search_url = f"https://api.vk.com/method/users.search?q={search_query}&count=10&fields=photo_max,career,education,status,last_seen&access_token=&v=5.199"
                # Требуется токен для полноценного поиска
                vk_results.append({
                    'source': 'vk_name_search',
                    'query': search_query,
                    'note': 'Требуется API токен для выполнения'
                })
            except Exception as e:
                print(f"    VK name search error: {e}")
        
        # Метод 4: Парсинг публичной страницы поиска
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
                    import re
                    profile_urls = re.findall(r'id\d+', response.text)
                    if profile_urls:
                        vk_results.append({
                            'source': 'vk_public_search',
                            'profiles': list(set(profile_urls))[:5],
                            'method': 'web_parse'
                        })
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
                    import re
                    profile_links = re.findall(r'profile/(\d+)', response.text)
                    if profile_links:
                        ok_results.append({
                            'source': 'ok_name_search',
                            'profiles': list(set(profile_links))[:5]
                        })
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
        
        # Поиск по телефону через API (требуется авторизация)
        if self.phone_e164:
            # Проверка через публичные сервисы
            try:
                # Используем сервис проверки регистрации
                check_url = f"https://api.pwrtelegram.xyz/check?phone={self.phone_e164}"
                response = requests.get(check_url, timeout=5)
                if response.status_code == 200:
                    tg_results.append({
                        'source': 'telegram_api_check',
                        'data': response.json(),
                        'method': 'public_api'
                    })
            except:
                pass
            
            tg_results.append({
                'source': 'telegram_phone_candidate',
                'phone': self.phone_e164,
                'note': 'Для проверки используйте поиск в Telegram по номеру'
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
        
        self.results['social_media']['instagram'] = ig_results
        return ig_results
    
    def search_data_leaks(self):
        """Поиск в базах утечек"""
        print("[*] Поиск в базах данных утечек...")
        
        leaks = []
        
        # Проверка email в haveibeenpwned
        if self.email:
            try:
                email_hash = hashlib.sha1(self.email.lower().encode()).hexdigest().upper()
                url = f"https://haveibeenpwned.com/api/v3/breachedaccount/{email_hash}"
                headers = {'hibp-api-key': 'ВАШ_КЛЮЧ_API'}  # Заменить на реальный ключ
                response = requests.get(url, headers=headers, timeout=5)
                if response.status_code == 200:
                    leaks.append({
                        'source': 'haveibeenpwned',
                        'breaches': response.json()
                    })
                elif response.status_code == 404:
                    leaks.append({
                        'source': 'haveibeenpwned',
                        'status': 'No breaches found'
                    })
            except Exception as e:
                print(f"    HIBP error: {e}")
        
        # Проверка через leak-lookup (если есть ключ)
        if self.email or self.phone_e164:
            try:
                # Демо-запрос (требуется регистрация)
                params = {'key': 'ВАШ_КЛЮЧ'}
                if self.email:
                    params['type'] = 'email_address'
                    params['query'] = self.email
                elif self.phone_e164:
                    params['type'] = 'phone'
                    params['query'] = self.phone_e164
                
                # Раскомментировать при наличии ключа
                # response = requests.get('https://leak-lookup.com/api/search', params=params)
                # if response.status_code == 200:
                #     leaks.append({
                #         'source': 'leak-lookup',
                #         'data': response.json()
                #     })
                
                leaks.append({
                    'source': 'leak-lookup',
                    'note': 'Требуется API ключ для доступа'
                })
            except Exception as e:
                print(f"    Leak lookup error: {e}")
        
        # Поиск в открытых базах (примеры)
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
            'databases': public_databases
        })
        
        self.results['data_leaks'] = leaks
        return leaks
    
    def search_public_records(self):
        """Поиск в публичных реестрах и базах"""
        print("[*] Поиск в публичных реестрах...")
        
        records = []
        
        # ФССП (исполнительные производства)
        if self.full_name and self.dob:
            try:
                url = "https://api.fssp.gov.ru/api/v1/search"
                params = {
                    'region': 'all',
                    'lastname': self.last_name,
                    'firstname': self.first_name,
                    'middlename': self.middle_name,
                    'birthdate': self.dob.replace('.', '')
                }
                # Требуется токен ФССП
                records.append({
                    'source': 'fssp',
                    'params': params,
                    'note': 'Требуется авторизация на портале ФССП'
                })
            except Exception as e:
                print(f"    FSSP error: {e}")
        
        # ЕГРЮЛ/ЕГРИП (налоговая)
        if self.full_name:
            try:
                # Используем публичный API nalog.ru
                url = "https://egrul.nalog.ru/"
                data = {
                    'vyp3CaptchaToken': '',
                    'page': '',
                    'query': f"{self.last_name} {self.first_name} {self.middle_name}"
                }
                response = requests.post(url, data=data, timeout=5)
                if response.status_code == 200:
                    # Парсинг результата
                    import re
                    task_id = re.search(r'<task>(.*?)</task>', response.text)
                    if task_id:
                        result_url = f"https://egrul.nalog.ru/search-result/{task_id.group(1)}"
                        records.append({
                            'source': 'egrul',
                            'url': result_url,
                            'method': 'search_initiated'
                        })
            except Exception as e:
                print(f"    EGRUL error: {e}")
        
        # Росреестр (недвижимость)
        records.append({
            'source': 'rosreestr',
            'note': 'Поиск по ФИО через портал Росреестра (требуется авторизация)'
        })
        
        # Суды (ГАС Правосудие)
        if self.full_name:
            try:
                url = "https://sudrf.ru/index.php"
                params = {
                    'last_name': self.last_name,
                    'first_name': self.first_name,
                    'middle_name': self.middle_name
                }
                records.append({
                    'source': 'sudrf',
                    'params': params,
                    'note': 'Поиск по делам в судах общей юрисдикции'
                })
            except Exception as e:
                print(f"    Sudrf error: {e}")
        
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
            
            if self.middle_name:
                nicknames.append(f"{self.first_name.lower()}{self.middle_name[0].lower()}{self.last_name.lower()}")
            
            associated['nickname_candidates'] = nicknames
        
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
        
        if self.phone:
            phone_clean = re.sub(r'\D', '', self.phone)
            if len(phone_clean) >= 7:
                passwords.append(phone_clean[-7:])
                passwords.append(phone_clean[-6:])
        
        associated['password_candidates'] = list(set(passwords))[:20]
        
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
                    for profile in item['profiles']:
                        likely_accounts.append({
                            'platform': 'vk',
                            'url': f"https://vk.com/{profile}"
                        })
        
        # Telegram кандидаты
        if 'telegram' in self.results['social_media'] and self.results['social_media']['telegram']:
            for item in self.results['social_media']['telegram']:
                if 'username' in item:
                    likely_accounts.append({
                        'platform': 'telegram',
                        'url': item['url']
                    })
        
        summary['likely_accounts'] = likely_accounts
        summary['password_candidates'] = self.results['associated_data'].get('password_candidates', [])
        
        report['summary'] = summary
        return report
    
    def save_report(self, filename=None):
        """Сохранение отчета в JSON"""
        report = self.generate_report()
        
        if not filename:
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            filename = f"osint_report_{self.last_name}_{self.first_name}_{timestamp}.json"
        
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
