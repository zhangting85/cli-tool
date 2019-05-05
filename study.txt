import time
import paramiko
import socket

import glob
import os


class SSHConnection(AbstractBaseConnection):
    conn_protocol = 'SSH'

    def __init__(self, host, port, timeout):
        port = int(port) if port else 22
        paramiko.util.log_to_file('./paramiko.log')
        self.client = paramiko.SSHClient()
        self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        _log = paramiko.util.get_logger('paramiko')
        for handler in _log.handlers:
            handler.close()
            _log.removeHandler(handler)
        self.host = host
        self.port = port
        self.channel = None
        self._prompt = None
        self._match_prompt = ''
        self.term = 'linux'
        self.width = 1024
        self.height = 512
        self.set_timeout(timeout)

    def _login(self, username, password):
        paramiko.util.log_to_file('./paramiko.log')
        self.username = username
        self.password = password
        try:
            self.client.connect(self.host, self.port, username, password)
        except paramiko.AuthenticationException:
            raise RuntimeError('%s Authentication Failure' % self.host)
        finally:
            _log = paramiko.util.get_logger('paramiko')
            for handler in _log.handlers:
                handler.close()
                _log.removeHandler(handler)

    def _close(self):
        try:
            if self.channel:
                self.channel.send('exit\n')
        except socket.error:
            pass
        self.channel = None
        self.client.close()
        print
        '*INFO* Disconnect from %s' % self.host

    def write(self, text):
        if self.channel is None:
            self.channel = self.client.invoke_shell(term=self.term, width=self.width, height=self.height)
            self.channel.set_combine_stderr(True)
        try:
            print
            text
            self.channel.sendall(text + self._newline)
        except socket.error:
            raise EOFError

    def read(self):
        data = ''
        if self.channel is None:
            self.channel = self.client.invoke_shell(term=self.term, width=self.width, height=self.height)
        self.channel.set_combine_stderr(True)
        while self.channel.recv_ready():
            _raw = self.channel.recv(1000)
            if _raw == '':
                time.sleep(0.001)
                continue
            data += _raw
        return data

    def _reconnect(self):
        self.channel = None
        self.client.close()
        del (self.client)
        self.client = paramiko.SSHClient()
        self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        self.client.connect(self.host, self.port, self.username, self.password)
        self.read_until_command_prompt()

    def get_file(self, remotepath, localpath, callback=None):
        self.sftp = paramiko.SFTPClient.from_transport(self.client._transport)
        return self.sftp.get(remotepath, localpath, callback)

    def put_file(self, source, destination, callback=None, confirm=True):
        mode = 00777  # stat.S_IRWXU | stat.S_IRWXG | stat.S_IRWXO
        filelist = glob.glob(source)
        dest_dir, dest_file = os.path.split(destination)
        if len(filelist) > 1 and dest_file != '':
            raise ValueError, 'Can\'t copy multiple source files to one destination file!'
        print
        "*INFO* Copying %s to %s" % (filelist, dest_dir)
        if dest_dir.count('\\') > 0:
            dirnames = dest_dir.replace('\\', '/').split('/')
        else:
            dirnames = dest_dir.split('/')
        sftp_client = self.client.open_sftp()
        for dirname in dirnames:
            if dirname == '' or dirname.count(':') > 0:
                sftp_client.chdir('/')
                continue
            else:
                try:
                    sftp_client.chdir(dirname)
                except IOError:
                    sftp_client.mkdir(dirname)
                    sftp_client.chdir(dirname)
        cwd = sftp_client.getcwd()
        for singlefile in filelist:
            if len(filelist) == 1 and dest_file != '' and os.path.isfile(singlefile):
                sftp_client.put(singlefile, cwd + '/' + dest_file)
                sftp_client.chmod(cwd + '/' + dest_file, mode)
                print
                "*INFO* Copied file from '%s'\n to '%s'" % (singlefile, cwd + '/' + dest_file)
            elif os.path.isfile(singlefile):
                sftp_client.put(singlefile, cwd + '/' + os.path.split(singlefile)[1])
                sftp_client.chmod(cwd + '/' + os.path.split(singlefile)[1], mode)
                print
                "*INFO* Copied file from '%s'\n to '%s'" % (singlefile, cwd + '/' + os.path.split(singlefile)[1])




class AbstractBaseConnection(object):
    _answer_prompt = '\[[Yy]/[Nn]\]: | yes or no: |n]: |[Pp]assword:\s+$|(yes/no)|\[[Yy]/n\]:|\[y/n\]|Y/N."|~$'

    _shell_prompt = '\[\S+@.*\]\s*[$#]\s*$|bash\-.*[$#]\s*$|lim-bash[$#]\s*$|\w+\-\d+:\~?.*\$\s*$|\w+@[\w-]+:\~?.*[$#]\s*'
    _scli_prompt = '.*@.*\s+\[.*\]\s+>\s*$'
    _custom_prompt = ''
    _encoding = 'utf8'
    _newline = '\n'
    _timeout = 10
    _console_type = ''

    def __init__(self, host='', port='', prompt='', timeout=''):
        pass

    def _login(self, username, password):
        raise_xxxError('xxxKeywordSyntaxError', 'The Function has been not implemented yet')

    def _close(self):
        raise_XXXError('xxxKeywordSyntaxError', 'The Function has been not implemented yet')

    def write(self, command):
        raise_XXXError('xxxKeywordSyntaxError', 'The Function has been not implemented yet')

    def read(self):
        raise_XXXError('xxxKeywordSyntaxError', 'The Function has been not implemented yet')

    def _reconnect(self):
        raise NotImplementedError

    def _encode(self, text):
        if isinstance(text, str):
            return text
        if not isinstance(text, basestring):
            text = unicode(text)
        return text.encode(self._encoding)

    def _decode(self, text):
        text = re.sub('\x1B\[([0-9]{1,2}(;[0-9]{1,2})?)?[m|K|C]?', '', text)
        try:
            return text.decode(self._encoding, 'ignore')
        except UnicodeEncodeError:
            return text

    def read_until(self, expected):
        data = ''
        max_time = time.time() + self._timeout
        while time.time() < max_time:
            _raw = self._decode(self.read())
            if _raw != '':
                data += _raw
                continue
            if expected in data:
                print '*INFO*', data
                return data
        print '*INFO*', data
        raise_XXXError('xxxCommandExecuteError',
                       'Read the output with expected string(%s) timeout!' % expected)

    def read_until_command_prompt(self):
        data = ''
        _shell_prompt_ptn = re.compile(self._shell_prompt)
        _scli_prompt_ptn = re.compile(self._scli_prompt)
        _answer_prompt_ptn = re.compile(self._answer_prompt)

        console_ptn_type_map = {_scli_prompt_ptn: 'scli',
                                _shell_prompt_ptn: 'shell',
                                _answer_prompt_ptn: ''}
        if self._custom_prompt:
            _custom_prompt_ptn = re.compile(self._custom_prompt)
            console_ptn_type_map[_custom_prompt_ptn] = ''
        max_time = self._timeout + time.time()

        while time.time() < max_time:
            _raw = self._decode(self.read())
            if _raw == '':
                continue

            if _raw.endswith('--More--'):
                data = data.rstrip() + '\n' + _raw.replace('--More--', '').strip()
                self.write(' ')
                continue

            data += _raw

            search_data = data[-1024:]
            for ptn in console_ptn_type_map:
                if ptn.search(search_data):
                    if console_ptn_type_map[ptn]:
                        self._console_type = console_ptn_type_map[ptn]
                    print '*INFO*', data
                    return data
        print '*INFO*', data
        raise AssertionError("No match found for prompt" + self._shell_prompt + '|' + self._scli_prompt + '|' + self._answer_prompt + '|' + self._custom_prompt)

    def connect_to_hardware(self, host, port, username, password, timeout):
        port = int(port)
        self._timeout = timestr_to_secs(timeout)
        self._login(username, password)
        return self.read_until_command_prompt()

    def disconnect_from_hardware(self):
        self._close()

    def _remove_prompt(self, output):
        prompt_pool = [self._answer_prompt, self._scli_prompt, self._shell_prompt, self._custom_prompt]
        return re.sub('|'.join(filter(lambda item: item, prompt_pool)), '', output)

    def execute_command(self, command, *answers):
        print '*TRACE* entering execute_command'
        prompt_pool = [self._scli_prompt, self._shell_prompt, self._custom_prompt]
        end_matcher = re.compile('|'.join(filter(lambda item: item, prompt_pool)))

        print '*INFO* Execute command:', command
        self.write(command)
        try:
            result = ""
            index = 0
            max_time = self._timeout + time.time()
            answer_len = len(answers)
            while time.time() < max_time:
                #result += self.read_until_command_prompt()
                tmp = self.read_until_command_prompt()
                if tmp.strip() == command.strip() or not tmp.strip():
                    continue
                result += tmp
                if end_matcher.search(result):
                    if command[:5] not in result:
                        continue
                    tmp_str = self._decode(result).replace(command, '',1)
                    return self._remove_prompt(tmp_str)

                if index == answer_len:
                    self.write('n')
                    continue
                self.write(answers[index])
                index += 1
        except EOFError:
            raise
        except:
            print result
            print '*WARN* execute_command:\n', traceback.format_exc()
            out = self.read()
            if not out:
                try:
                    self.write(chr(3))
                    time.sleep(1)
                    self.read_until_command_prompt()
                except:
                    print "*ERROR* connection has some problem"
                    raise EOFError
            while out != '':
                time.sleep(0.2)
                result += out
                out = self.read()

            return result

    def start_command(self,command):
        logging.info('*INFO* Execute command: {}', command)
        self.write(command)

    def put_file(self):
        raise NotImplementedError

    def get_file(self):
        raise NotImplementedError

    def set_timeout(self, timeout=''):
        if timeout:
            timeout, self._timeout = self._timeout, timestr_to_secs(timeout)
        else:
            timeout = secs_to_timestr(self._timeout)
        return timeout

    def set_prompt(self, prompt):
        prompt_pool = [self._answer_prompt, self._shell_prompt, self._scli_prompt, self._custom_prompt]
        old_prompt = '|'.join(filter(lambda item: item, prompt_pool))
        (prompt, self._custom_prompt) = (self._custom_prompt, prompt)
        return old_prompt

    def set_newline(self, newline):
        self._newline = newline.upper().replace('LF', '\n').replace('CR', '\r')

    def reconnect_connection(self):
        self._close()
        # add this sleep as a workaround for a pronto: user "_xxxadmin" could not login when reconnecting, before some service get started
        # time.sleep(120)
        self._reconnect()

    @property
    def console_type(self):
        return self._console_type

    def read_until_no_messages(self):
        data = ''
        max_time = self._timeout + time.time()
        retry = False
        while time.time() < max_time:
            _raw = self.read()
            if _raw != '':
                data += _raw
                continue
            else:
                if not retry:
                    retry = True
                    time.sleep(0.5)
                    continue
                else:
                    print data
                    data = self._decode(data)
                    return data
        raise AssertionError("Read_until_no_msg Time Out")
