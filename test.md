/workspaces/files-to-prompt/files_to_prompt/__init__.py
```python

```
/workspaces/files-to-prompt/files_to_prompt/__main__.py
```python
from .cli import cli

if __name__ == "__main__":
    cli()

```
/workspaces/files-to-prompt/files_to_prompt/cli.py
````python
import os
import sys
from fnmatch import fnmatch

import click
import json
import ast
from fsspec import filesystem
from fsspec.utils import get_protocol

global_index = 1

EXT_TO_LANG = {
    "py": "python",
    "c": "c",
    "cpp": "cpp",
    "java": "java",
    "js": "javascript",
    "ts": "typescript",
    "html": "html",
    "css": "css",
    "xml": "xml",
    "json": "json",
    "yaml": "yaml",
    "yml": "yaml",
    "sh": "bash",
    "rb": "ruby",
}


def read_storage_options(
    value: str | None = None,
) -> list | dict | None:
    """
    Parse dictionary or from various input formats.

    Supports:
    - JSON string
    - Python literal (dict/list)
    - Comma-separated key=value pairs (for dicts)
    - Comma-separated values (for lists)
    - List-like string with or without quotes

    Args:
        value (str, optional): Input string to parse

    Returns:
        dict | None: Parsed parameter or None if parsing fails
    """

    def convert_string_booleans(obj):
        if isinstance(obj, dict):
            return {k: convert_string_booleans(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [convert_string_booleans(item) for item in obj]
        elif isinstance(obj, str):
            if obj.lower() == "true":
                return True
            elif obj.lower() == "false":
                return False
        return obj

    if value is None:
        return None

    try:
        parsed = json.loads(value)
        return convert_string_booleans(parsed)
    except json.JSONDecodeError:
        try:
            parsed = ast.literal_eval(value)

            if not isinstance(parsed, dict):
                raise ValueError(f"Expected dict, got {type(parsed)}")
           
            return convert_string_booleans(parsed)

        except (SyntaxError, ValueError):
            if  "=" in value:
                parsed = dict(
                    pair.split("=", 1) for pair in value.split(",") if pair.strip()
                )
                return convert_string_booleans(parsed)

            return None


def should_ignore(path, gitignore_rules):
    for rule in gitignore_rules:
        if fnmatch(os.path.basename(path), rule):
            return True
        if os.path.isdir(path) and fnmatch(os.path.basename(path) + "/", rule):
            return True
    return False


def read_gitignore(path, fs):
    gitignore_path = os.path.join(path, ".gitignore")
    if os.path.isfile(gitignore_path):
        with fs.open(gitignore_path, "r") as f:
            return [
                line.strip() for line in f if line.strip() and not line.startswith("#")
            ]
    return []


def add_line_numbers(content):
    lines = content.splitlines()

    padding = len(str(len(lines)))

    numbered_lines = [f"{i + 1:{padding}}  {line}" for i, line in enumerate(lines)]
    return "\n".join(numbered_lines)


def print_path(writer, path, content, cxml, markdown, line_numbers):
    if cxml:
        print_as_xml(writer, path, content, line_numbers)
    elif markdown:
        print_as_markdown(writer, path, content, line_numbers)
    else:
        print_default(writer, path, content, line_numbers)


def print_default(writer, path, content, line_numbers):
    writer(path)
    writer("---")
    if line_numbers:
        content = add_line_numbers(content)
    writer(content)
    writer("")
    writer("---")


def print_as_xml(writer, path, content, line_numbers):
    global global_index
    writer(f'<document index="{global_index}">')
    writer(f"<source>{path}</source>")
    writer("<document_content>")
    if line_numbers:
        content = add_line_numbers(content)
    writer(content)
    writer("</document_content>")
    writer("</document>")
    global_index += 1


def print_as_markdown(writer, path, content, line_numbers):
    lang = EXT_TO_LANG.get(path.split(".")[-1], "")
    # Figure out how many backticks to use
    backticks = "```"
    while backticks in content:
        backticks += "`"
    writer(path)
    writer(f"{backticks}{lang}")
    if line_numbers:
        content = add_line_numbers(content)
    writer(content)
    writer(f"{backticks}")


def process_path(
    path,
    fs,
    extensions,
    include_hidden,
    ignore_files_only,
    ignore_gitignore,
    gitignore_rules,
    ignore_patterns,
    writer,
    claude_xml,
    markdown,
    line_numbers=False,
):
    if fs.isfile(path):
        try:
            with fs.open(path, "r") as f:
                print_path(writer, path, f.read(), claude_xml, markdown, line_numbers)
        except UnicodeDecodeError:
            warning_message = f"Warning: Skipping file {path} due to UnicodeDecodeError"
            click.echo(click.style(warning_message, fg="red"), err=True)
    elif fs.isdir(path):
        for root, dirs, files in fs.walk(path):
            if not include_hidden:
                dirs[:] = [d for d in dirs if not d.startswith(".")]
                files = [f for f in files if not f.startswith(".")]

            if not ignore_gitignore:
                gitignore_rules.extend(read_gitignore(root, fs))
                dirs[:] = [
                    d
                    for d in dirs
                    if not should_ignore(os.path.join(root, d), gitignore_rules)
                ]
                files = [
                    f
                    for f in files
                    if not should_ignore(os.path.join(root, f), gitignore_rules)
                ]

            if ignore_patterns:
                if not ignore_files_only:
                    dirs[:] = [
                        d
                        for d in dirs
                        if not any(fnmatch(d, pattern) for pattern in ignore_patterns)
                    ]
                files = [
                    f
                    for f in files
                    if not any(fnmatch(f, pattern) for pattern in ignore_patterns)
                ]

            if extensions:
                files = [f for f in files if f.endswith(extensions)]

            for file in sorted(files):
                file_path = os.path.join(root, file)
                try:
                    with fs.open(file_path, "r") as f:
                        print_path(
                            writer,
                            file_path,
                            f.read(),
                            claude_xml,
                            markdown,
                            line_numbers,
                        )
                except UnicodeDecodeError:
                    warning_message = (
                        f"Warning: Skipping file {file_path} due to UnicodeDecodeError"
                    )
                    click.echo(click.style(warning_message, fg="red"), err=True)


def read_paths_from_stdin(use_null_separator):
    if sys.stdin.isatty():
        # No ready input from stdin, don't block for input
        return []

    stdin_content = sys.stdin.read()
    if use_null_separator:
        paths = stdin_content.split("\0")
    else:
        paths = stdin_content.split()  # split on whitespace
    return [p for p in paths if p]


@click.command()
@click.argument("paths", nargs=-1, type=click.Path(exists=True))
@click.option("extensions", "-e", "--extension", multiple=True)
@click.option(
    "--include-hidden",
    is_flag=True,
    help="Include files and folders starting with .",
)
@click.option(
    "--ignore-files-only",
    is_flag=True,
    help="--ignore option only ignores files",
)
@click.option(
    "--ignore-gitignore",
    is_flag=True,
    help="Ignore .gitignore files and include all files",
)
@click.option(
    "ignore_patterns",
    "--ignore",
    multiple=True,
    default=[],
    help="List of patterns to ignore",
)
@click.option(
    "output_file",
    "-o",
    "--output",
    type=click.Path(writable=True),
    help="Output to a file instead of stdout",
)
@click.option(
    "claude_xml",
    "-c",
    "--cxml",
    is_flag=True,
    help="Output in XML-ish format suitable for Claude's long context window.",
)
@click.option(
    "markdown",
    "-m",
    "--markdown",
    is_flag=True,
    help="Output Markdown with fenced code blocks",
)
@click.option(
    "line_numbers",
    "-n",
    "--line-numbers",
    is_flag=True,
    help="Add line numbers to the output",
)
@click.option(
    "--null",
    "-0",
    is_flag=True,
    help="Use NUL character as separator when reading from stdin",
)
@click.option(
    "storage_options",
    "--storage-options",
    "-so",
    help="Storage options for the fsspec filesystem"
)
@click.version_option()
def cli(
    paths,
    extensions,
    include_hidden,
    ignore_files_only,
    ignore_gitignore,
    ignore_patterns,
    output_file,
    claude_xml,
    markdown,
    line_numbers,
    null,
    storage_options
):
    """
    Takes one or more paths to files or directories and outputs every file,
    recursively, each one preceded with its filename like this:

    \b
        path/to/file.py
        ----
        Contents of file.py goes here
        ---
        path/to/file2.py
        ---
        ...

    If the `--cxml` flag is provided, the output will be structured as follows:

    \b
        <documents>
        <document path="path/to/file1.txt">
        Contents of file1.txt
        </document>
        <document path="path/to/file2.txt">
        Contents of file2.txt
        </document>
        ...
        </documents>

    If the `--markdown` flag is provided, the output will be structured as follows:

    \b
        path/to/file1.py
        ```python
        Contents of file1.py
        ```
    """
    # Reset global_index for pytest
    global global_index
    global_index = 1

    # Read paths from stdin if available
    stdin_paths = read_paths_from_stdin(use_null_separator=null)

    # Combine paths from arguments and stdin
    paths = [*paths, *stdin_paths]

    gitignore_rules = []
    writer = click.echo
    fp = None
    if output_file:
        fp = open(output_file, "w", encoding="utf-8")
        writer = lambda s: print(s, file=fp)
    for path in paths:
        if not os.path.exists(path):
            raise click.BadArgumentUsage(f"Path does not exist: {path}")
        
        if claude_xml and path == paths[0]:
            writer("<documents>")
        if storage_options:
            protocol=get_protocol(path)
            fs = filesystem(protocol=protocol **read_storage_options)
            path = path.split("://")[-1]
        else:
            fs = filesystem("local")
        if not ignore_gitignore:
            gitignore_rules.extend(read_gitignore(os.path.dirname(path),fs))
        process_path(
            path,
            fs,
            extensions,
            include_hidden,
            ignore_files_only,
            ignore_gitignore,
            gitignore_rules,
            ignore_patterns,
            writer,
            claude_xml,
            markdown,
            line_numbers,
        )
    if claude_xml:
        writer("</documents>")
    if fp:
        fp.close()

````
/workspaces/files-to-prompt/files_to_prompt/fs.py
```python
import base64
import inspect
import os
import posixpath
import urllib
from pathlib import Path
from typing import Any

import fsspec
import requests
from fsspec import filesystem
from fsspec.implementations.cache_mapper import AbstractCacheMapper
from fsspec.implementations.cached import SimpleCacheFileSystem
from fsspec.implementations.dirfs import DirFileSystem
from fsspec.implementations.memory import MemoryFile
from fsspec.utils import infer_storage_options




from fsspec import AbstractFileSystem


class GitLabFileSystem(AbstractFileSystem):
    """FSSpec-compatible filesystem interface for GitLab repositories.

    Provides access to files in GitLab repositories through the GitLab API,
    supporting read operations with authentication.

    Attributes:
        project_name (str): Name of the GitLab project
        project_id (str): ID of the GitLab project
        access_token (str): GitLab personal access token
        branch (str): Git branch to read from
        base_url (str): GitLab instance URL

    Example:
        >>> # Access public project
        >>> fs = GitLabFileSystem(
        ...     project_name="my-project",
        ...     access_token="glpat-xxxx"
        ... )
        >>>
        >>> # Read file contents
        >>> with fs.open("path/to/file.txt") as f:
        ...     content = f.read()
        >>>
        >>> # List directory
        >>> files = fs.ls("path/to/dir")
        >>>
        >>> # Access enterprise GitLab
        >>> fs = GitLabFileSystem(
        ...     project_id="12345",
        ...     access_token="glpat-xxxx",
        ...     base_url="https://gitlab.company.com",
        ...     branch="develop"
        ... )
    """

    def __init__(
        self,
        project_name: str | None = None,
        project_id: str | None = None,
        access_token: str | None = None,
        branch: str = "main",
        base_url: str = "https://gitlab.com",
        **kwargs,
    ):
        """Initialize GitLab filesystem.

        Args:
            project_name: Name of the GitLab project. Required if project_id not provided.
            project_id: ID of the GitLab project. Required if project_name not provided.
            access_token: GitLab personal access token for authentication.
                Required for private repositories.
            branch: Git branch to read from. Defaults to "main".
            base_url: GitLab instance URL. Defaults to "https://gitlab.com".
            **kwargs: Additional arguments passed to AbstractFileSystem.

        Raises:
            ValueError: If neither project_name nor project_id is provided
            requests.RequestException: If GitLab API request fails
        """
        super().__init__(**kwargs)
        self.project_name = project_name
        self.project_id = project_id
        self.access_token = access_token
        self.branch = branch
        self.base_url = base_url.rstrip("/")
        self._validate_init()
        if not self.project_id:
            self.project_id = self._get_project_id()

    def _validate_init(self) -> None:
        """Validate initialization parameters.

        Ensures that either project_id or project_name is provided.

        Raises:
            ValueError: If neither project_id nor project_name is provided
        """
        if not self.project_id and not self.project_name:
            raise ValueError("Either 'project_id' or 'project_name' must be provided")

    def _get_project_id(self) -> str:
        """Retrieve project ID from GitLab API using project name.

        Makes an API request to search for projects and find the matching project ID.

        Returns:
            str: The GitLab project ID

        Raises:
            ValueError: If project not found
            requests.RequestException: If API request fails
        """
        url = f"{self.base_url}/api/v4/projects"
        headers = {"PRIVATE-TOKEN": self.access_token}
        params = {"search": self.project_name}
        response = requests.get(url, headers=headers, params=params)

        if response.status_code == 200:
            projects = response.json()
            for project in projects:
                if project["name"] == self.project_name:
                    return project["id"]
            raise ValueError(f"Project '{self.project_name}' not found")
        else:
            response.raise_for_status()

    def _open(self, path: str, mode: str = "rb", **kwargs) -> MemoryFile:
        """Open a file from GitLab repository.

        Retrieves file content from GitLab API and returns it as a memory file.

        Args:
            path: Path to file within repository
            mode: File open mode. Only "rb" (read binary) is supported.
            **kwargs: Additional arguments (unused)

        Returns:
            MemoryFile: File-like object containing file content

        Raises:
            NotImplementedError: If mode is not "rb"
            requests.RequestException: If API request fails

        Example:
            >>> fs = GitLabFileSystem(project_id="12345", access_token="glpat-xxxx")
            >>> with fs.open("README.md") as f:
            ...     content = f.read()
            ...     print(content.decode())
        """
        if mode != "rb":
            raise NotImplementedError("Only read mode is supported")

        url = (
            f"{self.base_url}/api/v4/projects/{self.project_id}/repository/files/"
            f"{urllib.parse.quote_plus(path)}?ref={self.branch}"
        )
        headers = {"PRIVATE-TOKEN": self.access_token}
        response = requests.get(url, headers=headers)

        if response.status_code == 200:
            file_content = base64.b64decode(response.json()["content"])
            return MemoryFile(None, None, file_content)
        else:
            response.raise_for_status()

    def _ls(self, path: str, detail: bool = False, **kwargs) -> list[str] | list[dict]:
        """List contents of a directory in GitLab repository.

        Args:
            path: Directory path within repository
            detail: Whether to return detailed information about each entry.
                If True, returns list of dicts with file metadata.
                If False, returns list of filenames.
            **kwargs: Additional arguments (unused)

        Returns:
            list[str] | list[dict]: List of file/directory names or detailed info

        Raises:
            requests.RequestException: If API request fails

        Example:
            >>> fs = GitLabFileSystem(project_id="12345", access_token="glpat-xxxx")
            >>> # List filenames
            >>> files = fs.ls("docs")
            >>> print(files)
            ['README.md', 'API.md']
            >>>
            >>> # List with details
            >>> details = fs.ls("docs", detail=True)
            >>> for item in details:
            ...     print(f"{item['name']}: {item['type']}")
        """
        url = f"{self.base_url}/api/v4/projects/{self.project_id}/repository/tree?path={path}&ref={self.branch}"
        headers = {"PRIVATE-TOKEN": self.access_token}
        response = requests.get(url, headers=headers)

        if response.status_code == 200:
            files = response.json()
            if detail:
                return files
            else:
                return [file["name"] for file in files]
        else:
            response.raise_for_status()


try:
    fsspec.register_implementation("gitlab", GitLabFileSystem)
except ValueError as e:
    _ = e


def get_filesystem(
    path: str | Path | None = None,
    storage_options: dict[str, str] | None = None,
   
) -> AbstractFileSystem:
    """Get a filesystem instance based on path or configuration.

    This function creates and configures a filesystem instance based on the provided path
    and options. It supports various filesystem types including local, S3, GCS, Azure,
    and Git-based filesystems.

    Args:
        path: URI or path to the filesystem location. Examples:
            - Local: "/path/to/data"
            - S3: "s3://bucket/path"
            - GCS: "gs://bucket/path"
            - Azure: "abfs://container/path"
            - GitHub: "github://org/repo/path"
        storage_options: Configuration options for the filesystem. Can be:
            - Dictionary of key-value pairs for authentication/configuration
            - None to use environment variables or default credentials
        dirfs: Whether to wrap filesystem in DirFileSystem for path-based operations.
            Set to False when you need direct protocol-specific features.
        cached: Whether to enable local caching of remote files.
            Useful for frequently accessed remote files.
        cache_storage: Directory path for cached files. Defaults to path-based location
            in current directory if not specified.
        fs: Existing filesystem instance to wrap with caching or dirfs.
            Use this to customize an existing filesystem instance.
        **storage_options_kwargs: Additional keyword arguments for storage options.
            Alternative to passing storage_options dictionary.

    Returns:
        AbstractFileSystem: Configured filesystem instance with requested features.

    Raises:
        ValueError: If storage protocol or options are invalid
        FSSpecError: If filesystem initialization fails
        ImportError: If required filesystem backend is not installed

    Example:
        >>> # Local filesystem
        >>> fs = get_filesystem("/path/to/data")
        >>>
        >>> # S3 with credentials
        >>> fs = get_filesystem(
        ...     "s3://bucket/data",
        ...     storage_options={
        ...         "key": "ACCESS_KEY",
        ...         "secret": "SECRET_KEY"
        ...     }
        ... )
        >>>
        >>> # Cached GCS filesystem
        >>> fs = get_filesystem(
        ...     "gs://bucket/data",
        ...     storage_options{
        ...         token="service_account.json"
        ...     },
        ...     cached=True,
        ...     cache_storage="/tmp/gcs_cache"
        ... )

        >>>
        >>> # Wrap existing filesystem
        >>> base_fs = filesystem("s3", key="ACCESS", secret="SECRET")
        >>> cached_fs = get_filesystem(
        ...     fs=base_fs,
        ...     cached=True
        ... )
    """

    protocol = get_protocol(path)

    fs = fsspec.filesystem(protocol=protocol, **storage_options)

    return fs
```
