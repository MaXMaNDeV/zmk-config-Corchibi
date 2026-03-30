# zephyr_library()


#---
# Patch 1: Skip CONFIGURATION_RESET on startup (causes I2C issues on nRF52840)
# Patch 2: Movement accumulator with threshold and timeout
#          - Accumulates small movements, reports when threshold exceeded
#          - Resets accumulator if no movement for 100ms
#          - Prevents typing vibration from triggering auto-mouse
#          - Preserves all intentional trackball movement
#---
set(FILE_PATH "${ZEPHYR_BASE}/drivers/input/input_pat912x.c")
file(READ "${FILE_PATH}" FILE_CONTENT)

# Patch 1: Skip CONFIGURATION_RESET write
string(
  REPLACE "\ti2c_reg_write_byte_dt(&cfg->i2c, PAT912X_CONFIGURATION, CONFIGURATION_RESET);\n"
          "\t\/\/ i2c_reg_write_byte_dt(&cfg->i2c, PAT912X_CONFIGURATION, CONFIGURATION_RESET);\n"
  FILE_CONTENT "${FILE_CONTENT}"
)

# Patch 2: Add accumulator variables after pat912x_data struct usage
string(
  REPLACE "static void pat912x_motion_work_handler(struct k_work *work)\n{"
          "static int pat912x_acc_x = 0;\nstatic int pat912x_acc_y = 0;\nstatic int64_t pat912x_acc_last_time = 0;\n\nstatic void pat912x_motion_work_handler(struct k_work *work)\n{"
  FILE_CONTENT "${FILE_CONTENT}"
)

# Patch 2b: Add accumulator logic before input_report_rel
string(
  REPLACE "\tLOG_DBG(\"x=%4d y=%4d\", x, y);\n\n\tif (cfg->axis_x >= 0) {"
          "\tLOG_DBG(\"x=%4d y=%4d\", x, y);\n\n\t\/\/ Movement accumulator (threshold=5, timeout=100ms)\n\t{\n\t\tint64_t now = k_uptime_get();\n\t\tif (now - pat912x_acc_last_time > 50) {\n\t\t\tpat912x_acc_x = 0;\n\t\t\tpat912x_acc_y = 0;\n\t\t}\n\t\tpat912x_acc_last_time = now;\n\t\tpat912x_acc_x += x;\n\t\tpat912x_acc_y += y;\n\t\tif (abs(pat912x_acc_x) + abs(pat912x_acc_y) <= 3) {\n\t\t\tk_work_submit(&data->motion_work);\n\t\t\treturn;\n\t\t}\n\t\tx = pat912x_acc_x;\n\t\ty = pat912x_acc_y;\n\t\tpat912x_acc_x = 0;\n\t\tpat912x_acc_y = 0;\n\t}\n\n\tif (cfg->axis_x >= 0) {"
  FILE_CONTENT "${FILE_CONTENT}"
)

file(WRITE "${FILE_PATH}" "${FILE_CONTENT}")
message(STATUS "Patched PAT912x driver (reset skip + movement accumulator): ${FILE_PATH}")


# zephyr_include_directories(include)
