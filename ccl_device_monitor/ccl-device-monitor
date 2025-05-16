import re
import logging
import subprocess
import sys
import time
import os
from argparse import ArgumentParser
import yaml
from pathlib import Path
import pandas as pd
import multiprocessing
from functools import partial
from jnpr.junos import Device
from datetime import datetime

# Global variables
SYSFILES_FOLDER = "ccl_labs_sysfiles/"
CCL_LABS_FOLDER = "/homes/svinukonda/ccl_labs/"
TOOL_NAME = "ccl_device_monitor"

# Ensure sysfiles folder exists
subprocess.run(f"mkdir -p {SYSFILES_FOLDER}", shell=True)

def main_heading(str1, print_flag=True):
    """Print a formatted heading."""
    len1 = len(str1)
    print_str = f"\n ==>> ***********{'*' * len1}***********"
    print_str += f"\n ==>> *** {' ' * len1}               ***"
    print_str += f"\n ==>> *** <<||   {str1}   ||>> ***"
    print_str += f"\n ==>> *** {' ' * len1}               ***"
    print_str += f"\n ==>> ***********{'*' * len1}***********"
    if print_flag:
        print(print_str)
    return print_str

def check_file_exists(path):
    """Check if a file exists."""
    result = subprocess.run(f'ls -lt {path}', shell=True, capture_output=True, text=True)
    return not bool(re.search("No such file or directory", result.stderr))

def regexp_on_match(regexp, mat2, match_all):
    """Perform regex matching on input string or list."""
    if isinstance(mat2, list):
        mat2 = "\n".join(mat2)
    if isinstance(regexp, list):
        for v in regexp:
            mat2 = regexp_on_match(v, mat2, match_all)
        return mat2
    if match_all:
        return re.findall(regexp, mat2)
    return re.search(regexp, mat2)

def execute_command_in_linux_shell(cmd):
    """Execute a command in the Linux shell and return output."""
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    return result.stdout + result.stderr

def perform_test_checks_on_device_core(dev, title, cmd_dict, capstr=""):
    """Core function to perform test checks on a device."""
    ret_lst = []
    if "cmd" in cmd_dict or "command" in cmd_dict:
        mode = cmd_dict.get("mode", "cli")
        cmd = cmd_dict.get("cmd") or cmd_dict.get("command")
        if mode == "cli":
            try:
                capstr = dev.cli(cmd, warning=False).strip()
            except Exception:
                ret_lst.append([title, "**ERROR1**"])
                return ret_lst
        elif mode == "linux":
            capstr = execute_command_in_linux_shell(cmd).strip()

    flag_dict_val_present = False
    for key, val in cmd_dict.items():
        if isinstance(val, dict):
            flag_dict_val_present = True
            ret = perform_test_checks_on_device_core(dev, key, cmd_dict[key], capstr)
            ret_lst.extend(ret)

    if not flag_dict_val_present:
        match_all = cmd_dict.get("match_all", False)
        regexp = cmd_dict.get("regexp")
        record = cmd_dict.get("record")
        match = capstr if regexp is None else regexp_on_match(regexp, capstr, match_all)
        if record is None:
            ret_lst.append([title, match])
        else:
            try:
                value = eval(record)
                ret_lst.append([title, value])
            except Exception:
                ret_lst.append([title, "**ERROR2**"])
    return ret_lst

def append_dict_to_csv(csv_file, new_row):
    """Append a dictionary as a row to a CSV file."""
    try:
        df = pd.read_csv(csv_file)
    except FileNotFoundError:
        df = pd.DataFrame(columns=new_row.keys())
    df = pd.concat([df, pd.DataFrame([new_row])], ignore_index=True)
    df.to_csv(csv_file, index=False)
    print("Row appended successfully.")

def perform_test_checks_on_device(dev, device_health_test_dict, output_csv_file):
    """Perform health checks on a device and save to CSV."""
    ret_lst_main = []
    for k, v in device_health_test_dict.items():
        if isinstance(v, dict):
            main_heading(k)
            if "rpc_cmd" in v:
                try:
                    match = eval(f"dev.rpc.{v['rpc_cmd']}")
                    if "record" not in v:
                        ret_lst_main.append([k, match])
                    else:
                        ret3 = eval(v["record"])
                        ret_lst_main.append([k, ret3])
                except Exception:
                    ret_lst_main.append([k, "**ERROR3**"])
            elif "cmd" in v or "command" in v:
                ret_lst = perform_test_checks_on_device_core(dev, k, v)
                ret_lst_main.extend(ret_lst)
                print("\n\n")

    if not os.path.exists(output_csv_file):
        column_lst = [lst[0] for lst in ret_lst_main]
        pd.DataFrame(columns=column_lst).to_csv(output_csv_file, index=False)

    ret_dict = {lst[0]: lst[1] for lst in ret_lst_main}
    append_dict_to_csv(output_csv_file, ret_dict)

def record_device_health(dev_ssh_info, monitor_yaml_file, output_csv_file, monitoring_timeout, loop_sleeptime=30):
    """Record device health metrics over time."""
    dev_ssh = Device(host=dev_ssh_info[0], user=dev_ssh_info[1], password=dev_ssh_info[2], port=22)
    try:
        dev_ssh.open()
    except Exception as err:
        logging.error(f'Exception: Cannot connect to device: {err}')
        return -1

    start_time = time.time()
    while True:
        elapsed_time = time.time() - start_time
        if elapsed_time > monitoring_timeout:
            print("Monitoring timeout reached. Exiting loop.")
            break

        monitor_dict = yaml.safe_load(Path(monitor_yaml_file).read_text())
        try:
            perform_test_checks_on_device(dev_ssh, monitor_dict["monitor"], output_csv_file)
        except Exception:
            record_device_health(dev_ssh_info, monitor_yaml_file, output_csv_file, monitoring_timeout - elapsed_time, loop_sleeptime)
            break

        time.sleep(loop_sleeptime)
        print(f"|**TEST**| Remaining Time/Total Time: {int(elapsed_time)}/{int(monitoring_timeout)}")

    dev_ssh.close()
    print("Loop exited.")

def my_background_function(stop_event, func_name, *args, **kwargs):
    """Run a function in a background process."""
    print(f"Background process started with PID: {os.getpid()}")
    func_name(*args, **kwargs)
    print(f"Background process (PID: {os.getpid()}) finished execution.")
    stop_event.set()

def start_background_process(func_name, *args, **kwargs):
    """Start a background process and return process and stop event."""
    stop_event = multiprocessing.Event()
    process = multiprocessing.Process(target=partial(my_background_function, stop_event, func_name, *args, **kwargs))
    process.daemon = False
    process.start()
    return process, stop_event

def USER_INPUT_TEMPLATE1(desc, input_dict, toolname="TOOL"):
    """Prompt user for input with options and return selected key."""
    main_heading(desc)
    while True:
        input_msg = "\n\t==> OPTIONS: "
        flag_default_variable_check = False
        for key in sorted(input_dict.keys()):
            input_msg += f"\n\t\t > {key}: {input_dict[key]}"
            if "(default)" in input_dict[key]:
                flag_default_variable_check = True
                default_key = key
        input_msg += "\n\t\t > q: quit"
        input_msg += "\n\n\t==> ENTER YOUR CHOICE: "
        ip = input(input_msg)

        if ip == "" and flag_default_variable_check:
            return default_key
        if ip.lower().startswith('q'):
            print("\n")
            main_heading(f"THANK YOU FOR USING {toolname}")
            sys.exit()
        if ip and ip[0] in input_dict:
            return ip[0]

def create_public_link(source_dir, folder_in_public_html, symbolic_link_name):
    """Create a public symbolic link for the output directory."""
    user_name = os.environ['USER']
    os.system(f"chmod -R 755 ~/public_html")
    os.system(f"mkdir -p ~/public_html/{folder_in_public_html}")
    os.system(f"chmod -R 755 ~/public_html/{folder_in_public_html}")
    os.system(f"cd ~/public_html/{folder_in_public_html}; ln -sfn {source_dir} {symbolic_link_name}")
    link_name = f"https://ttsv-web01.juniper.net/~{user_name}/{folder_in_public_html}/{symbolic_link_name}"
    main_heading(f"Created Link: {link_name}")
    return link_name

if __name__ == '__main__':
    main_heading("CCL-DEVICE-MONITOR")
    parser = ArgumentParser(description="Monitor devices and save metrics to CSV.")
    parser.add_argument('file', type=str, nargs='?', help="Input YAML file for device monitoring")
    args = parser.parse_args()

    user_input_file = args.file or "user_input_ccl_device_monitor.yaml"
    user_input_file_with_path = SYSFILES_FOLDER + user_input_file if not args.file else user_input_file

    if not check_file_exists(user_input_file_with_path):
        os.system(f"cp {CCL_LABS_FOLDER}/{TOOL_NAME}/{user_input_file} {SYSFILES_FOLDER}/.")
        ch = USER_INPUT_TEMPLATE1(
            f"Input file {user_input_file_with_path} created.",
            {'y': 'Open Input File --(default)', 'c': 'Continue without opening'},
            TOOL_NAME
        )
        if ch == 'y':
            os.system(f"vi {user_input_file_with_path}")
    else:
        ch = USER_INPUT_TEMPLATE1(
            f"Input file {user_input_file_with_path} already exists.",
            {'y': 'Open existing Input File --(default)', 'r': 'Reload default and open', 'c': 'Continue without opening'},
            TOOL_NAME
        )
        if ch == 'r':
            os.system(f"cp {CCL_LABS_FOLDER}/{TOOL_NAME}/{user_input_file} {SYSFILES_FOLDER}/.")
            os.system(f"vi {user_input_file_with_path}")
        elif ch == 'y':
            os.system(f"vi {user_input_file_with_path}")

    monitor_dict = yaml.safe_load(Path(user_input_file_with_path).read_text())
    dev_info_lst = monitor_dict["framework_variables"]["dev_info_list"]
    monitoring_time = monitor_dict["framework_variables"]["monitoring_time"]
    loop_sleeptime = monitor_dict["framework_variables"]["loop_sleeptime"]
    output_file_substr = monitor_dict["framework_variables"]["output_file_substr"]

    output_dir = f"ccl_device_monitor_output/ccl_device_monitor__{datetime.now().strftime('%Y%m%d_%H%M%S')}"
    os.system(f"mkdir -p {output_dir}")

    bg_process_info_lst = []
    for dev_info in dev_info_lst:
        dev_info = [x.strip() for x in dev_info.split(",")]
        bg_process, bg_stop_event = start_background_process(
            record_device_health, dev_info, user_input_file_with_path,
            f"{output_dir}/ccl_device_monitor_output_{dev_info[0]}.csv", monitoring_time, loop_sleeptime
        )
        bg_process_info_lst.append([bg_process, bg_stop_event])

    start_time = time.time()
    while True:
        elapsed_time = time.time() - start_time
        flag_to_stay = False
        for process_info in bg_process_info_lst:
            if not process_info[1].is_set():
                flag_to_stay = True

        if not flag_to_stay:
            main_heading("All processes are Done")
            break
        if elapsed_time <= monitoring_time:
            main_heading("Processes are Running")
            main_heading(f"|** ccl-device-monitor **| Remaining Time (in Total Time): {int(elapsed_time)}/{monitoring_time}")
            time.sleep(60)
        else:
            main_heading("Processes are Running. But monitoring_time is completed. Hence exiting.")
            break

    for dev_info in dev_info_lst:
        dev_info = [x.strip() for x in dev_info.split(",")]
        os.system(f"/homes/svinukonda/ccl_labs/ccl-convert-csv-to-html {output_dir}/ccl_device_monitor_output_{dev_info[0]}.csv")
    os.system(f"chmod -R 755 {output_dir}")
    create_public_link(output_dir, "ccl_device_monitor_output", output_dir.rsplit("/", 1)[-1])