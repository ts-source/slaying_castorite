# slaying_castorite

  if [[ -n "$TEAM_NEXT_PAGE" ]]; then
    TEAM_DATA=$(gh api graphql -f query="$QUERY" -F owner="$OWNER" -F pageSize="$REPO_PAGE_SIZE" -F endCursor="$TEAM_NEXT_PAGE")
  else
    TEAM_DATA=$(gh api graphql -f query="$QUERY" -F owner="$OWNER" -F pageSize="$REPO_PAGE_SIZE")
  fi

  ERROR_CODE=$?

  if [ "${ERROR_CODE}" -ne 0 ]; then
    echo "Error getting Teams for Org: ${OWNER}"
    echo "${TEAM_DATA}"
  else
    ERROR_MESSAGE=$(echo "$TEAM_DATA" | jq -r '.errors[]?')

    if [[ -n "${ERROR_MESSAGE}" ]]; then
      echo "ERROR --- Errors occurred while retrieving teams for org: ${OWNER}"
      echo "${ERROR_MESSAGE}" | jq '.'
    fi

    TEAMS=$(echo "${TEAM_DATA}" | jq '.data.organization.teams.nodes')


    ##########################
    # Get the Next Page Flag #
    ##########################
    HAS_NEXT_TEAM_PAGE=$(echo "${TEAM_DATA}" | jq -r '.data.organization.teams.pageInfo.hasNextPage')

    ##############################
    # Get the Current End Cursor #
    ##############################
    TEAM_NEXT_PAGE=$(echo "${TEAM_DATA}" | jq -r '.data.organization.teams.pageInfo.endCursor')

    for TEAM in $(echo -n "${TEAMS}" | jq -r '.[] | @base64'); do
      _team_jq() {
        echo -n "${TEAM}" | base64 --decode | jq -r "${1}"
      }

      TEAM_NAME=$(_team_jq '.slug')

      ### Check the team name against array of all previously-processed teams
      TEAM_INDEX=-1

      for i in "${!TEAM_LIST[@]}"; do
      if [[ "${TEAM_LIST[$i]}" = "${TEAM_NAME}" ]]; then
        TEAM_INDEX=${i}
      fi
      done

      ### If this is the first instance of that team name, add it to the list and add the org name to its array
      if [[ ${TEAM_INDEX} -eq -1 ]]; then
        Debug "Team: ${TEAM_NAME} is unique. Adding to the list!"
        TEAM_LIST+=( "${TEAM_NAME}" )
        TEAM_ORG_LIST[(( ${#TEAM_LIST[@]} - 1 ))]=${ORG_NAME}
        NUMBER_OF_TEAM_CONFLICTS[(( ${#TEAM_LIST[@]} - 1 ))]=1
      else
        Debug "Team: $TEAM_NAME already exists. Adding ${ORG_NAME} to the conflict list"
        TEAM_ORG_LIST[${TEAM_INDEX}]+=" ${ORG_NAME}"
        (( NUMBER_OF_TEAM_CONFLICTS[TEAM_INDEX]++ ))
      fi
    done

    ########################################
    # See if we need to loop for more data #
    ########################################
    if [ "${HAS_NEXT_TEAM_PAGE}" == "false" ]; then
      # We have all the data, we can move on
      Debug "Gathered all teams from PR"
      TEAM_NEXT_PAGE=""
    elif [ "${HAS_NEXT_TEAM_PAGE}" == "true" ]; then
      # We need to loop through GitHub to get all teams
      Debug "More pages of teams. Gathering next batch."

      #######################################
      # Call GetTeams with new cursor #
      #######################################
      GetTeams
    else
      # Failing to get this value means we didnt get a good response back from GitHub
      # And it could be bad input from user, not enough access, or a bad token
      # Fail out and have user validate the info
      echo ""
      echo "######################################################"
      echo "ERROR! Failed response back from GitHub!"
      echo "Please validate your PAT, Organization, and access levels!"
      echo "######################################################"
      exit 1
    fi
  fi
}
################################################################################
#### Function MarkMigrationIssues ##############################################
MarkMigrationIssues() {
  # Need to read the output files, and total the issues and see
  # if over 60k objects or repo is over 2gb

  ##############
  # Read Input #
  ##############
  REPO_SIZE="$1"
  RECORD_COUNT="$2"

  # Check if more than 60k objects
  if [ "${RECORD_COUNT}" -ge 60000 ] || [ "${REPO_SIZE}" -gt 1500 ]; then
    echo "0"
    return 0
  else
    echo "1"
    return 1
  fi
}
################################################################################
#### Function ReportConflicts ##################################################
ReportConflicts() {
  if [[ ${ANALYZE_CONFLICTS} -eq 1 ]]; then
    for (( i=0; i<${#REPO_LIST[@]}; i++)) do
      if (( ${NUMBER_OF_CONFLICTS[$i]} > 1 )); then
        echo "${NUMBER_OF_CONFLICTS[$i]},${REPO_LIST[$i]},${GROUP_LIST[$i]}" >> "${REPO_CONFLICTS_OUTPUT_FILE}"
      fi
    done
  fi

  ##########################
  # Check to analyze teams #
  ##########################
  if [[ ${ANALYZE_TEAMS} -eq 1 ]]; then
    for (( i=0; i<${#TEAM_LIST[@]}; i++)) do
      if (( ${NUMBER_OF_TEAM_CONFLICTS[$i]} > 1 )); then
        echo "${NUMBER_OF_TEAM_CONFLICTS[$i]},${TEAM_LIST[$i]},${TEAM_ORG_LIST[$i]}" >> "${TEAM_CONFLICTS_OUTPUT_FILE}"
      fi
    done
  fi
}
################################################################################
#### Function ConvertKBToMB ####################################################
ConvertKBToMB() {
  ####################################
  # Value that needs to be converted #
  ####################################
  VALUE=$1

  ##############################
  # Validate that its a number #
  ##############################
  REGEX='^[0-9]+$'
  if ! [[ ${VALUE} =~ ${REGEX} ]] ; then
    echo "ERROR! Not a number:[${VALUE}]"
    exit 1
  fi

  #################
  # Convert to MB #
  #################
  SIZEINMB=$((VALUE/1024))
  echo "${SIZEINMB}"

  ####################
  # Return the value #
  ####################
  return ${SIZEINMB}
}
################################################################################
#### Function ValidateJQ #######################################################
ValidateJQ() {
  # Need to validate the machine has jq installed as we use it to do the parsing
