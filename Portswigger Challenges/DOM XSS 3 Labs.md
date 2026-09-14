import argparse
import json
import re
from urllib.parse import urljoin, urlparse, parse_qs

import requests
from bs4 import BeautifulSoup


class WebMapper:
    def __init__(self, target, max_pages=20):
        self.target = self.normalize_url(target)
        self.base_domain = urlparse(self.target).netloc

        self.max_pages = max_pages

        self.visited = set()
        self.pages = set()
        self.links = set()
        self.external_links = set()
        self.parameters = set()
        self.forms = []
        self.scripts = set()
        self.endpoints = set()
        self.robots = []

        self.session = requests.Session()

        self.session.headers.update({
            "User-Agent": "WebMapper/2.0"
        })

    # ---------------------------------------------------------
    # URL FUNCTIONS
    # ---------------------------------------------------------

    def normalize_url(self, url):
        if not url.startswith(("http://", "https://")):
            url = "https://" + url

        return url.rstrip("/")

    def is_same_domain(self, url):
        return urlparse(url).netloc == self.base_domain

    # ---------------------------------------------------------
    # HTTP
    # ---------------------------------------------------------

    def request_page(self, url):
        try:
            response = self.session.get(
                url,
                timeout=10,
                allow_redirects=True
            )

            print(
                f"[+] {response.status_code:<3} "
                f"{response.url}"
            )

            return response

        except requests.RequestException as error:
            print(f"[-] Request failed: {url}")
            print(f"    {error}")

            return None

    # ---------------------------------------------------------
    # HTML LINK EXTRACTION
    # ---------------------------------------------------------

    def extract_links(self, soup, current_url):

        for tag in soup.find_all("a", href=True):

            href = tag.get("href").strip()

            if not href:
                continue

            full_url = urljoin(current_url, href)

            parsed = urlparse(full_url)

            if parsed.scheme not in ("http", "https"):
                continue

            # Remove fragments
            clean_url = parsed._replace(
                fragment=""
            ).geturl()

            if self.is_same_domain(clean_url):

                self.links.add(clean_url)

                if parsed.query:
                    self.extract_parameters(clean_url)

            else:
                self.external_links.add(clean_url)

    # ---------------------------------------------------------
    # PARAMETERS
    # ---------------------------------------------------------

    def extract_parameters(self, url):

        parsed = urlparse(url)

        parameters = parse_qs(
            parsed.query,
            keep_blank_values=True
        )

        for parameter in parameters:
            self.parameters.add(parameter)

    # ---------------------------------------------------------
    # FORMS
    # ---------------------------------------------------------

    def extract_forms(self, soup, current_url):

        for form in soup.find_all("form"):

            action = form.get("action", "")

            method = form.get(
                "method",
                "GET"
            ).upper()

            action_url = urljoin(
                current_url,
                action
            )

            fields = []

            for field in form.find_all(
                ["input", "textarea", "select"]
            ):

                name = field.get("name")

                if name:
                    fields.append({
                        "name": name,
                        "type": field.get(
                            "type",
                            field.name
                        )
                    })

            form_data = {
                "url": action_url,
                "method": method,
                "fields": fields
            }

            if form_data not in self.forms:
                self.forms.append(form_data)

    # ---------------------------------------------------------
    # JAVASCRIPT
    # ---------------------------------------------------------

    def extract_scripts(self, soup, current_url):

        for script in soup.find_all("script"):

            src = script.get("src")

            if src:

                script_url = urljoin(
                    current_url,
                    src
                )

                self.scripts.add(script_url)

            else:

                # Analyze inline JavaScript
                code = script.get_text()

                self.extract_endpoints_from_text(
                    code,
                    current_url
                )

    # ---------------------------------------------------------
    # ENDPOINT DISCOVERY
    # ---------------------------------------------------------

    def extract_endpoints_from_text(
        self,
        text,
        current_url
    ):

        # URLs beginning with /
        paths = re.findall(
            r"""["'`](/[^"'`\s<>]+)["'`]""",
            text
        )

        for path in paths:

            if path.startswith("//"):
                continue

            endpoint = urljoin(
                current_url,
                path
            )

            parsed = urlparse(endpoint)

            # Avoid collecting normal static files
            if parsed.path.lower().endswith(
                (
                    ".jpg",
                    ".jpeg",
                    ".png",
                    ".gif",
                    ".svg",
                    ".css",
                    ".woff",
                    ".woff2",
                    ".ttf",
                    ".ico"
                )
            ):
                continue

            self.endpoints.add(endpoint)

        # Look for fetch()
        fetch_matches = re.findall(
            r"""fetch\(\s*["'`]([^"'`]+)""",
            text
        )

        for endpoint in fetch_matches:

            self.endpoints.add(
                urljoin(current_url, endpoint)
            )

        # Look for jQuery AJAX URLs
        ajax_matches = re.findall(
            r"""url\s*:\s*["'`]([^"'`]+)""",
            text
        )

        for endpoint in ajax_matches:

            self.endpoints.add(
                urljoin(current_url, endpoint)
            )

    # ---------------------------------------------------------
    # JAVASCRIPT FILE ANALYSIS
    # ---------------------------------------------------------

    def analyze_script(self, script_url):

        try:

            response = self.session.get(
                script_url,
                timeout=10
            )

            if response.status_code != 200:
                return

            print(f"    [JS] {script_url}")

            self.extract_endpoints_from_text(
                response.text,
                script_url
            )

        except requests.RequestException:
            pass

    # ---------------------------------------------------------
    # ROBOTS.TXT
    # ---------------------------------------------------------

    def get_robots(self):

        robots_url = urljoin(
            self.target + "/",
            "robots.txt"
        )

        try:

            response = self.session.get(
                robots_url,
                timeout=10
            )

            if response.status_code == 200:

                for line in response.text.splitlines():

                    line = line.strip()

                    if line.lower().startswith(
                        ("allow:", "disallow:")
                    ):
                        self.robots.append(line)

        except requests.RequestException:
            pass

    # ---------------------------------------------------------
    # CRAWLER
    # ---------------------------------------------------------

    def crawl(self):

        queue = [self.target]

        while queue and len(
            self.visited
        ) < self.max_pages:

            current_url = queue.pop(0)

            if current_url in self.visited:
                continue

            if not self.is_same_domain(
                current_url
            ):
                continue

            self.visited.add(current_url)

            response = self.request_page(
                current_url
            )

            if response is None:
                continue

            content_type = response.headers.get(
                "Content-Type",
                ""
            )

            if "text/html" not in content_type:
                continue

            self.pages.add(response.url)

            soup = BeautifulSoup(
                response.text,
                "html.parser"
            )

            self.extract_links(
                soup,
                response.url
            )

            self.extract_forms(
                soup,
                response.url
            )

            self.extract_scripts(
                soup,
                response.url
            )

            # Add discovered pages to queue
            for link in self.links:

                if (
                    link not in self.visited
                    and link not in queue
                    and self.is_same_domain(link)
                ):
                    queue.append(link)

        self.get_robots()

        # Analyze discovered JavaScript
        for script in self.scripts:
            self.analyze_script(script)

    # ---------------------------------------------------------
    # OUTPUT
    # ---------------------------------------------------------

    def display_results(self):

        print("\n")
        print("=" * 60)
        print("              WEB APPLICATION MAP")
        print("=" * 60)

        print("\n[DOMAIN]")
        print(f"  {self.base_domain}")

        print("\n[PAGES]")
        for page in sorted(self.pages):
            print(f"  {page}")

        print("\n[LINKS]")
        for link in sorted(self.links):
            print(f"  {link}")

        print("\n[PARAMETERS]")
        for parameter in sorted(self.parameters):
            print(f"  ?{parameter}")

        print("\n[FORMS]")

        for form in self.forms:

            print(
                f"  {form['method']} "
                f"{form['url']}"
            )

            for field in form["fields"]:
                print(
                    f"      {field['name']}"
                    f" ({field['type']})"
                )

        print("\n[JAVASCRIPT]")

        for script in sorted(self.scripts):
            print(f"  {script}")

        print("\n[ENDPOINTS / API-LIKE]")

        for endpoint in sorted(self.endpoints):
            print(f"  {endpoint}")

        print("\n[EXTERNAL LINKS]")

        for link in sorted(self.external_links):
            print(f"  {link}")

        print("\n[ROBOTS.TXT]")

        if self.robots:

            for line in self.robots:
                print(f"  {line}")

        else:
            print("  No robots.txt rules found")

        print("\n[SUMMARY]")
        print(f"  Pages:           {len(self.pages)}")
        print(f"  Links:           {len(self.links)}")
        print(f"  Parameters:      {len(self.parameters)}")
        print(f"  Forms:            {len(self.forms)}")
        print(f"  JavaScript:       {len(self.scripts)}")
        print(f"  Endpoints:        {len(self.endpoints)}")
        print(f"  External links:   {len(self.external_links)}")

        print("=" * 60)

    # ---------------------------------------------------------
    # JSON EXPORT
    # ---------------------------------------------------------

    def save_json(self, filename):

        data = {
            "domain": self.base_domain,
            "pages": sorted(self.pages),
            "links": sorted(self.links),
            "parameters": sorted(self.parameters),
            "forms": self.forms,
            "javascript": sorted(self.scripts),
            "endpoints": sorted(self.endpoints),
            "external_links": sorted(
                self.external_links
            ),
            "robots": self.robots
        }

        with open(
            filename,
            "w",
            encoding="utf-8"
        ) as file:

            json.dump(
                data,
                file,
                indent=4
            )

        print(
            f"\n[+] Map saved to {filename}"
        )


# =============================================================
# MAIN
# =============================================================
def main():

    target = input("Enter target URL: ").strip()

    if not target.startswith(("http://", "https://")):
        target = "https://" + target

    mapper = WebMapper(
        target,
        max_pages=20
    )

    print(
        f"\n[+] Starting mapper against "
        f"{mapper.target}"
    )

    mapper.crawl()

    mapper.display_results()

    mapper.save_json("map.json")


if __name__ == "__main__":
    main()