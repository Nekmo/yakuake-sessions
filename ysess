#!/usr/bin/env python
# coding=utf-8

import subprocess as sp, os, math, sys, re
import subprocess
import time
from ConfigParser import ConfigParser
from cStringIO import StringIO
from optparse import OptionParser

import dbus

DBUS = 'qdbus org.kde.yakuake '
YAKUAKE_DEFAULT_NEW_SESSIONS_OPENED = 1

bus = dbus.SessionBus()


class SortedDict(dict):
    def __init__(self, *args, **kwargs):
        dict.__init__(self, *args, **kwargs)
        self.keyOrder = dict.keys(self)
    def keys(self):
        return self.keyOrder
    def iterkeys(self):
        for key in self.keyOrder:
            yield key
    __iter__ = iterkeys
    def items(self):
        return [(key, self[key]) for key in self.keyOrder]
    def iteritems(self):
        for key in self.keyOrder:
            yield (key, self[key])
    def values(self):
        return [self[key] for key in self.keyOrder]
    def itervalues(self):
        for key in self.keyOrder:
            yield self[key]
    def __setitem__(self, key, val):
        self.keyOrder.append(key)
        dict.__setitem__(self, key, val)
    def __delitem__(self, key):
        self.keyOrder.remove(key)
        dict.__delitem__(self, key)


def get_stdout(cmd, **opts):
    opts.update({'stdout': sp.PIPE})
    if 'env' in opts:
        env, opts['env'] = opts['env'], os.environ.copy()
        opts['env'].update(env)
    quoted = re.findall(r'".+"', cmd)
    for q in quoted:
        cmd = cmd.replace(q, '%s')
    cmd = cmd.split()
    for i, part in enumerate(cmd):
        if part == '%s':
            cmd[i] = quoted.pop(0)[1:-1]
    proc = sp.Popen(cmd, **opts)
    return proc.communicate()[0].strip()


def get_yakuake(cmd):
    return get_stdout(DBUS + cmd)


def get_sessions():
    tabs = []
    sessnum = len(get_yakuake('/yakuake/sessions terminalIdList').split(','))
    activesess = int(get_yakuake('/yakuake/sessions activeSessionId'))

    sessions = sorted(int(i) for i in get_yakuake('/yakuake/sessions sessionIdList').split(','))
    ksessions = sorted(int(line.split('/')[-1]) for line in get_yakuake('').split('\n') if '/Sessions/' in line)
    session_map = dict(zip(sessions, ksessions))
    last_tabid = None

    for ksession in ksessions:
        # Está dando la sesión del id en el que se hizo el split, no tiene nada que ver con el tab, pero ayuda.
        # Por ejemplo, sesión 4->split->sesión 5->split->sesión 6, dará sesión 5 para la sesión 6.
        # Al usar "sessionAtTab", me devuelve la primera sesión abierta en el tab, lo cual me ayuda a seguir
        # el orden.
        tabid = int(get_yakuake('/yakuake/sessions sessionIdForTerminalId %d' % (ksession - 1)))
        split = '' if tabid != last_tabid else ('vertical' if ksession % 2 else 'horizontal')
        last_tabid = tabid
        sessid = int(get_yakuake('/yakuake/tabs sessionAtTab %d' % tabid))
        # ksess = '/Sessions/%d' % session_map[sessid]
        ksess = '/Sessions/%d' % ksession
        pid = get_yakuake(ksess + ' processId')
        # cat /proc/<pid>/environ
        fgpid = get_yakuake(ksess+' foregroundProcessId')
        tabs.append({
            'title': get_yakuake('/yakuake/tabs tabTitle %d' % tabid),
            'sessionid': tabid,
            'tabid': tabid,
            'active': sessid == activesess,
            'split': split,
            'cwd': get_stdout('pwdx '+pid).partition(' ')[2],
            'cmd': '' if fgpid == pid else get_stdout('ps '+fgpid, env={'PS_FORMAT': 'command'}).split('\n')[-1],
        })
    return tabs


def format_sessions(tabs, fp):
    cp = ConfigParser(dict_type=SortedDict)
    tabpad = int(math.log10(len(tabs))) + 1
    for i, tab in enumerate(tabs):
        section = ('Tab %%0%dd' % tabpad) % (i+1)
        cp.add_section(section)
        cp.set(section, 'title', tab['title'])
        cp.set(section, 'active', 1 if tab['active'] else 0)
        cp.set(section, 'tab', tab['tabid'])
        cp.set(section, 'split', tab['split'])
        cp.set(section, 'cwd', tab['cwd'])
        cp.set(section, 'cmd', tab['cmd'])
    cp.write(fp)


def clear_sessions():
    ksessions = [line for line in get_yakuake('').split('\n') if '/Sessions/' in line]
    for ksess in ksessions:
        get_yakuake(ksess+' close')


def load_sessions(file):
    cp = ConfigParser(dict_type=SortedDict)
    cp.readfp(file)
    sections = cp.sections()
    if not sections:
        print >>sys.stderr, "No tab info found, aborting"
        sys.exit(1)

    # Clear existing sessions, but only if we have good info (above)
    # clear_sessions()
    subprocess.call(['killall', 'yakuake'])
    subprocess.call(['yakuake'])
    time.sleep(2)
    # for section in sections:
    #     get_yakuake('/yakuake/sessions addSession')
    # import time
    # time.sleep(5)

    # Map the new sessions to their konsole session objects
    # sessions = sorted(int(i) for i in get_yakuake('/yakuake/sessions sessionIdList').split(','))
    # ksessions = sorted(int(line.split('/')[-1]) for line in get_yakuake('').split('\n') if '/Sessions/' in line)
    # session_map = SortedDict(zip(sessions, ksessions))

    tab = 0
    active = 0
    # Repopulate the tabs
    for i, section in enumerate(sections):
        opts = dict(cp.items(section))
        if not opts['split']:
            tab += 1
            get_yakuake('/yakuake/sessions addSession')
            get_yakuake('/yakuake/tabs setTabTitle %d "%s"' % (tab, opts['title']))
        else:
            split_target, split = (tab, opts['split']) if ':' not in opts['split'] else opts['split'].split(':')
            get_yakuake('/yakuake/sessions splitTerminal{} {}'.format({'vertical': 'LeftRight',
                                                                       'horizontal': 'TopBottom'}[split],
                                                                      int(split_target)))
        sessid = int(get_yakuake('/yakuake/tabs sessionAtTab %d' % i))
        # ksessid = '/Session/%d' % session_map[sessid]
        if opts['cwd']:
            get_yakuake('/yakuake/sessions runCommand " cd %s"' % opts['cwd'])
        if opts['cmd']:
            for cmd in opts['cmd'].split(r'\n'):
                # get_yakuake('/yakuake/sessions runCommand "%s"' % cmd)
                dbus_session = bus.get_object('org.kde.yakuake', '/Sessions/{}'.format(i + 1))
                dbus_session = dbus.Interface(dbus_session, 'org.kde.konsole.Session')
                dbus_session.sendText(cmd)
                dbus_session.sendText('\n')
                # get_stdout('qdbus org.kde.yakuake /Sessions/%d org.kde.konsole.Session.sendText "%s"' % (i+1, cmd))
                # get_stdout('qdbus org.kde.yakuake /Sessions/{} org.kde.konsole.Session.sendText "\n"'.format(i+1))
        if opts['active'].lower() in ['y', 'yes', 'true', '1']:
            active = sessid
    if active:
        get_yakuake('/yakuake/sessions raiseSession %d' % active)
    # Remove initial session
    get_yakuake('/yakuake/sessions removeSession 0')


if __name__ == '__main__':
    # TODO: also store shell environment (for virtualenvs and such)
    # ps e 20017 | awk '{for (i=1; i<6; i++) $i = ""; print}'

    op = OptionParser(description="Save and load yakuake sessions.  Settings are exported in INI format.  Default action is to print the current setup to stdout in INI format.")
    op.add_option('-i', '--in-file', dest='infile', help='File to read from, or "-" for stdin', metavar='FILE')
    op.add_option('-o', '--out-file', dest='outfile', help='File to write to, or "-" for stdout', metavar='FILE')
    op.add_option('--force-overwrite', dest='force_overwrite', help='Do not prompt for confirmation if out-file exists', action="store_true", default=False)
    opts, args = op.parse_args()

    if opts.outfile is None and opts.infile is None:
        format_sessions(get_sessions(), sys.stdout)
    elif opts.outfile:
        fp = sys.stdout
        if opts.outfile and opts.outfile != '-' and (
            not os.path.exists(opts.outfile)
            or opts.force_overwrite
            or raw_input('Specified file exists, overwrite? [y/N] ').lower().startswith('y')):
            fp = open(opts.outfile, 'w')
        format_sessions(get_sessions(), fp)
    elif opts.infile:
        fp = sys.stdin
        if opts.infile and opts.infile != '-':
            if not os.path.exists(opts.infile):
                print >>sys.stderr, "ERROR: Input file (%s) does not exist." % opt.infile
                sys.exit(1)
            fp = open(opts.infile, 'r')
        load_sessions(fp)

